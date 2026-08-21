# PRD: AI 工作台定时任务 v1

> 状态：**草案（待终审，r2 修订）** — 2026-08-21
> 决策记录：Q1–Q21 已于 2026-08-21 两轮访谈确认（用户采纳全部推荐答案），本草案为烘焙后的完整版。
> 终审修订 r2：部署形态更正为**多实例共库**（推翻 Q1 单实例前提）——互斥机制改为 DB 行级原子抢占，跨实例同一任务至多一个执行（详见「调度与执行」1/3/5/6）。
> 前置母本：`plans/ai-chat-sse-prd.md`（AI Chat v1，聊天轮次落库结构）、`plans/chat-conversation-management-prd.md`（会话管理，type 过滤原决策）。

## Problem Statement

业务用户目前只能**守在屏幕前**手动发起 AI 对话：想要"每天早上 9 点汇总昨日订单异常""每周一生成上周采购周报"这类周期性工作，必须每天人肉登录、重复输入同样的 prompt。已有能力（SSE 聊天、会话管理）解决了"即时问答"与"历史回看"，但没有解决**"约定时间自动执行"**：

- 用户无法让系统在指定时间自动唤醒 agent 执行预设 prompt
- 周期性重复劳动无法沉淀为可配置、可启停、可复用的任务
- 执行产出没有与现有会话体系打通，无法沿用既有历史回看路径

## Solution

在 `ehub-ai-workbench` 服务中新增**定时任务模块**：用户创建任务（名称 + 预设 prompt + cron 表达式），系统内建调度器按约定时间自动唤醒百炼 agent 执行，产出直接落入任务绑定的专属会话组——用户回到现有会话历史即可查看每次执行结果。

- **任务配置**：创建/查询/更新/删除/启停/立即手动执行，六接口
- **到点自动执行**：各实例 `@Scheduled` 轮询任务表，到期任务经 DB 行级原子抢占（多实例下同一任务至多一个实例执行）后以非流式方式调用百炼
- **产出归档**：每次执行把 user/assistant 两行写入绑定会话组，复用现有会话内容查询接口回看
- **安全边界**：cron 间隔下限、每用户任务数上限、同任务严格串行、失败不重试只记录

本 PRD 仅覆盖后端接口；前端（ehub-web）UI 另行安排。

## User Stories

1. 作为业务用户，我想创建一个定时任务（预设 prompt + cron 时间），以便系统在约定时间自动执行而无需我在线操作
2. 作为业务用户，我想分页查看我的定时任务列表（含启停状态、最近执行时间与结局），以便掌握每个任务的运行情况
3. 作为业务用户，我想更新任务的名称/prompt/执行时间/启停状态，以便调整任务内容
4. 作为业务用户，我想删除不再需要的定时任务，以便清理我的任务列表
5. 作为业务用户，我想手动立即执行一次任务，以便在正式启用前验证 prompt 效果
6. 作为业务用户，我想在现有会话历史中查看定时任务每次执行的产出（含 token 用量），以便回顾结果
7. 作为业务用户，我想让我的定时任务及其产出只有我自己可见，以保证数据隔离

## Implementation Decisions

**总体策略**：在现有 chat/conversation 双域之外新增 `task` 领域；调度为各实例 `@Scheduled` DB 轮询 + 行级原子抢占（多实例安全），执行复用百炼 SDK 非流式调用，产出完全复用既有会话/内容表，仅新增一张任务表。

### 调度与执行

1. **调度基础设施（Q1，r2 修订）**：Spring `@Scheduled` 固定延迟轮询 DB 任务表（`taskPollFixedDelay`，默认 30s）。部署形态为**多实例共库**：每个实例都轮询，重复扫描无害——正确性不靠"只有一个调度者"，而靠下面的行级抢占保证，无需 Quartz/分布式锁/独立调度中心
2. **触发条件（Q9）**：轮询扫描 `enabled=1 AND is_del=0 AND next_fire_time <= now` 的任务，错过的多个触发点**合并为一次执行**；执行后 `next_fire_time` 推进为 cron 的下一个未来触发点
3. **跨实例互斥与防重入（Q11，r2 修订）**：抢占语义即"取到任务后立刻改状态、随即释放锁"——以单语句原子 UPDATE 落在触发行级锁上实现：`UPDATE ai_scheduled_task SET run_state='RUNNING', next_fire_time=<cron下一未来点> WHERE id=? AND run_state='IDLE'`。多实例同时到达时，InnoDB 对该行串行化：恰一个实例影响行数=1（抢到），其余=0（跳过）。该写法是 `SELECT ... FOR UPDATE` + 判断 + `UPDATE` + 提交释放的**折叠形式**——少一次往返、无需显式事务、锁仅持有单条语句时长，语义完全等价，v1 采纳单语句版；扫描 SELECT 本身不加锁（重复扫描无害）。调度轮询单线程，执行提交到独立任务线程池（core 2 / max 4），池满拒绝时记日志、复位 IDLE、该次触发放弃（`next_fire_time` 已推进，接受跳过）
4. **agent 调用形态（Q3）**：非流式——dashscope SDK `Application.call(param)` 一次取全量结果，复用现有 `appKey/appId` 配置；超时 `taskCallTimeout`（默认 120s）按 FAILED 处理
5. **僵死复位（Q11，r2 补充）**：`run_state=RUNNING` 超过 `runningStaleMinutes`（默认 30 分钟，可配）视为实例被杀残留，任意实例的轮询都可发起复位——复位同样走带 `run_state='RUNNING'` 守卫的原子 UPDATE，多实例同时复位至多一个成功；`runningStaleMinutes` 远大于 `taskCallTimeout`（30min ≫ 120s），不会误伤在途执行；复位不触发补跑（等下一个 `next_fire_time`）
6. **时钟（Q19，r2 补充）**：统一注入 `Clock` bean（生产系统时区，测试固定时钟）；cron 按 JVM 本地时区解释，不做用户级时区转换。多实例时钟偏差处理：到期比较（`next_fire_time <= now`）在 SQL 中用**数据库 `NOW()`**（全实例同一时间基准）；cron 下一触发点计算用应用 `Clock`，实例间秒级偏差可接受（不影响互斥，仅影响触发提前/延后数秒）

### 数据模型

7. **新表 `ai_scheduled_task`（Q6）**：`id/name/prompt/cron_expression/status(1=ENABLED,0=DISABLED)/run_state(IDLE/RUNNING)/conversation_id/next_fire_time/last_run_time/last_run_status(SUCCESS/FAILED)/last_run_error(摘要,≤500)` + 既有审计列（crt_/upd_，对齐 `BaseEntity`）；同时冗余**创建人 id+name 快照**（定时触发线程无 `UserContext`，以任务归属人身份落库，Q8）
8. **产出归属（Q4）**：任务创建时同步创建绑定会话组 `ai_conversation`，`type=SCHEDULED`（新增枚举值，数值取 **6**——同库老项目已占用 0/1/3/4/5，避开冲突），会话组 name=任务名；每次执行写 user/assistant 两行到 `ai_conversation_content`，`params` 标记 `{"trigger":"CRON|MANUAL","taskId":<id>}`；**不改动现有两张表**（来源标记走 params JSON）
9. **记忆连续性（Q5）**：绑定会话组持续复用同一个百炼 `sessionId`（现有回写机制），agent 记得该任务历史执行内容，适合"基于上周数据写周报"类任务；隔离开关为演进项

### 接口契约

10. **六接口（Q7）**，`/task` 前缀 + 统一响应包装（SuccessResponse/TablePageResponse）：
    - `POST /task`：创建，body `{name, prompt, cronExpression}`
    - `GET /task/page?page=&size=`：分页列表（默认 1/10，size≤50）
    - `PUT /task/{id}`：更新，body `{name, prompt, cronExpression, enabled}` **全量必填**（Q15）
    - `DELETE /task/{id}`：逻辑删除
    - `POST /task/{id}/run`：立即手动执行一次
    - `GET /task/{id}`：详情（含 conversationId）
11. **列表/详情字段（Q17）**：每用户 ≤20 的量级下列表返回全量：`id/name/prompt/cronExpression/enabled/conversationId/nextFireTime/lastRunTime/lastRunStatus/lastRunError/crtTime`；时间字段沿用 Long→String、日期序列化既有 Jackson 约定
12. **cron 约束（Q2/Q12）**：仅 Spring 6 段 cron 表达式；创建与更新时用 `CronExpression.next()` 算相邻两个未来触发点，**间隔 <5 分钟 → PARAM_ERROR**（防秒级高频打爆配额）
13. **配额（Q13）**：每用户任务数上限 20（含停用、不含已删），配置 `maxTasksPerUser`；超出 → PARAM_ERROR
14. **手动执行语义（Q14）**：停用任务也可手动执行（调试 prompt 用）；执行中再触发 → PARAM_ERROR "任务正在执行"；手动执行**不推进** `next_fire_time`；`params.trigger=MANUAL`
15. **更新语义（Q15）**：cron 变更或重新启用时重算 `next_fire_time`；停用时 `next_fire_time` 置 null；**执行中任务的更新对当前执行无效**（本次仍用旧 prompt），下一轮生效
16. **删除边界（Q16）**：逻辑删 `is_del=1`；正在执行的一轮**让它跑完**（产出照常落库）；已删任务不再被调度扫描；**绑定会话组及其内容保留**（审计/成本核算口径一致），仍可经会话内容接口访问
17. **归属校验与错误码（Q8/Q21）**：不存在/不属于当前用户/已删统一 `NOT_FOUND "任务不存在"`；校验失败 `PARAM_ERROR`（name ≤100 非空白、prompt ≤8000 复用现有上限）；`ErrorCodeEnum` 不新增枚举
18. **会话列表 type 过滤放开（Q18）**：顺带修正 `GET /chat/conversations` 实现中硬编码 `type=CHAT` 的差距，对齐管理 PRD 原决策——`type` 作为可选查询参数，不传返回该用户全部类型（CHAT+SCHEDULED）

### 已知边界（接受）

- 用户删除任务绑定的会话组后，任务继续执行、产出仍写入该（已逻辑删）会话组——对齐"删除不取消进行中 SSE 流"既有口径；若要求联动需另行评估
- 池满放弃触发、僵死复位不补跑：极端场景下损失一次执行，记日志可查
- 应用服务器间秒级时钟偏差：到期判定以 DB 时间为准，残余影响仅触发点提前/延后数秒

## Testing Decisions

**策略（Q20）**：四层组合，延续既有 mock 策略，不引入 Testcontainers：

1. **cron 校验单测（纯函数）**：合法/非法表达式；相邻触发间隔 <5 分钟拒绝
2. **调度触发单测（mock mapper + 固定 Clock）**：到期触发并推进 next_fire_time；占位 UPDATE 0 行跳过（该用例即跨实例互斥的机制验证——第二个"实例"再抢同一任务必得 0 行）；停用/已删不被扫描；RUNNING 超 30 分钟僵死复位
3. **执行服务单测（mock 百炼 Application.call）**：成功落库两行 + 更新 last_run_*；失败/超时记 FAILED + 错误摘要；落库以任务归属人快照身份写入
4. **MockMvc 集成测试（六端点各 1 条）**：创建校验（cron 非法/超配额）、列表分页、更新、删除、手动执行（含执行中冲突）、详情；断言统一响应壳与错误码

测试外部行为不测实现细节；prior art：ai-chat 的 mock SDK 单测 + MockMvc 模式。

## Out of Scope

- 前端 UI（ehub-web 任务管理页面，另行安排）
- 独立调度中心（Quartz 集群 / xxl-job / 分布式锁组件）——DB 行级原子抢占已满足多实例互斥，不引入
- 任务执行失败自动重试、执行历史流水表（仅任务表记最近一次）
- 任务级"每次独立新开记忆"开关（v1 固定连续记忆）
- 固定间隔、一次性指定时间两种触发方式（v1 仅 cron）
- 用户级时区转换（cron 按 JVM 时区）
- 任务执行结果的消息通知（钉钉/邮件推送）
- 对已逻辑删会话组的删除联动

## Further Notes

- **前置开发项**：`ErrorCodeEnum` 无 `PARAM_ERROR`（既有缺口，与 chat-conversation PRD 共用）；MyBatis-Plus 分页插件（同一前置）；本功能新增 `SCHEDULED(6)` 枚举值 + `Clock` bean + 任务线程池
- **与既有模块关系**：产出落库复用 `ai_conversation`/`ai_conversation_content` 与会话内容查询接口；不依赖 v1 SSE 流式链路（非流式独立调用）
- **残留小决策（终审确认项）**：SCHEDULED 枚举数值取 6；绑定会话组 name=任务名；删会话组后任务继续执行的已知边界
- **演进方向**：执行历史流水、失败重试策略、任务模板、通知推送、任务级记忆隔离开关
