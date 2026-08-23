# 变更日志（CHANGELOG）

本文件记录文档工程的全部重要变更，按日期倒序。格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## 2026-08-23

### Changed
- **设计产物按模块分目录**（`designs/` 结构化拆分）：平铺在 `designs/` 根目录的 9 个设计产物按模块归入三个子目录（与 `requirements/<feature>/` 对应）——`designs/ai-chat/`（模块设计）、`designs/chat-conversation/`（模块设计）、`designs/scheduled-task/`（模块设计 + 高保真原型 `-pages.pen` + 4 张 PNG 预览 + 状态机验证原型 `prototype-scheduled-task-state.html`）；`README.md` 目录树、文档流程产物落点约定（`designs/<feature>/<feature>-ia.md` 等）、文件名约定行、当前文档索引 Design 列同步更新；`specs/ai-chat-spec.md`、`specs/chat-conversation-spec.md`、`plans/chat-conversation-*-prd.md`、`requirements/scheduled-task/01|05` 交叉引用与设计文档内部相对链接（`../` → `../../`）同步修正。已跟踪文件经 `git mv` 保留历史。
- **创建领域模型文档**（`CONTEXT.md`）：统一记录 AI 工作台的会话、定时任务、任务执行轮次、执行状态、僵死任务及其关系、生命周期、不变量和关键业务场景；后续领域模型关系变更优先同步此文档，术语简明定义仍统一维护于 `GLOSSARY.md`。

## 2026-08-22

### Changed
- **定时任务 v1 PRD 定版**（`plans/scheduled-task-prd.md` v1.0）：用户终审通过，r3 修订随稿确认；三项残留小决策（SCHEDULED 枚举=6、绑定会话组 name=任务名、删会话组后任务继续执行边界）终审采纳。状态由"草案 r3 待终审"→"已定版 v1.0"；`requirements/scheduled-task/04` 母本引用与 README 索引同步。
- **定时任务 PRD r3 修订**：多实例负载分布不保证均匀定为 v1 接受边界——赛跑语义下忙实例不让位、闲实例不加成；v1 成本主体为百炼调用（全局计费，与执行实例无关），均匀性无收益。退化信号=「池满放弃触发」日志频率，出现即触发策略评估。
- **v2 计划排定**（PRD Further Notes）：执行历史流水表（另扩展表）→ 执行看板（实例偏斜度/池满放弃频率/触发延迟/成功率分布）→ **据看板数据评估**是否优化执行策略（随机相位、容量感知抢占）及失败重试；Out of Scope 与演进方向同步改写。同步更新 `requirements/scheduled-task/04` 母本引用（r2→r3）与 README 索引状态。

### Added
- **Skills 使用指南入 README**：新增「Skills 使用指南」章节——25 个 agent 技能的情境速查表（想法打磨/拆解落地/出问题/沟通知识/代码库健康五类情境 → 对应 skill 与说明）、与「文档流程」的咬合 mermaid 图（grill-with-docs → write-a-prd → to-spec → handoff → to-tickets/implement）及五条要点（仓库内拷问统一走 grill-with-docs、prototype 垫在交互草图前、to-spec 模板对齐 requirements 01–05、实现类 skill 在代码工程运行）。
- **文档流程 v2 定案（设计双关卡链路）**（`README.md`）：主线固化为 **PRD草案 → PRD定稿 → 正式PRD文档 →【产品交互设计】→【UI设计文档】→ 设计验收 → 开发实现**，两段【】展开为设计子链 **交互草图/信息架构 → 交互评审 → 高保真交互原型 → UI视觉设计 → 设计文档输出**；设两道关卡（①交互评审=FR 覆盖核对、②设计验收=覆盖+走查+齐套），评审/验收结论回填文档；新增两类设计产物落点（`-ia.md` 信息架构+评审结论、`-ui-spec.md` UI 设计文档），`-design.md` 增设设计验收小节；工程拆解与交互设计并行推进说明；mermaid 图改为 TD 双关卡结构，目录树/文件名约定/索引表说明同步。
- **文档流程全链路固化**（`README.md`）：「文档流程」显式固化为六步——PRD 草案（write-a-prd）→ 访谈定稿（grilling，interview 落盘）→ 最终 PRD v1.0（决策母本）→ requirements 拆解 → 设计文档（-design.md）→ **产品交互设计（-pages.pen + PNG 预览）** → 开发实现；mermaid 图同步加交互设计节点，目录树 designs/ 注释更新，索引表下补标准链路说明（定时任务 v1 已完整跑通，此前模块无交互设计属历史缺口）。
- **定时任务需求拆解文档全量落盘**（`requirements/scheduled-task/` 01/02/03/05，与既有 04 凑齐五件套）：01 产品概述（背景/方案/用户故事/术语）｜02 功能需求 FR-01~08（六接口 + FR-07 调度行为验收 + FR-08 会话列表 type 过滤实现差距修正 Q18）｜03 接口规范（`/task` 六接口契约、请求/响应示例、越权测试要求）｜05 非功能需求与风险（NFR-01~06：快照身份落库、池满放弃、DB NOW() 到期判定、可测试性；R-01~06：调度空窗、无重试、池满跳过、更新/删除竞态、token 成本）。README 目录树与索引行同步（"其余拆解中"→全量引用）。
- **定时任务模块设计文档**（`designs/scheduled-task-design.md` v1.0）：两深模块一复用（TaskService 六接口业务 / TaskExecutionService 执行生命周期 / saveChatResult 原样复用）；调度内部结构（僵死复位→扫描→单语句抢占→池满放弃→分发）、run_state×status 正交状态机与不变式、多实例抢占时序图（DB NOW() 基准）、三线程模型（HTTP/调度/执行，快照身份）、D1–D10 设计决策（溯源决策 N/Qxx/r2/r3）、四层测试映射、v2 演进缝（流水表/看板/重试/记忆隔离/负载均匀）。
- **定时任务页面交互设计**（`designs/scheduled-task-pages.pen` + 4 张 PNG 预览）：Element Plus 风格四页面——①任务列表（侧边导航+搜索/状态筛选+表格：名称/cron 双行、状态 tag、下次/最近执行、立即执行/编辑/删除+分页）②新建/编辑对话框（名称≤100、prompt≤8000 带字数统计、cron 预设每天/每周/每月/自定义+下次执行预览+5 分钟间隔规则提示、创建即启用开关）③立即执行确认（不影响原排程/手动标记来源/执行中不可触发）④删除确认（会话组保留）+ 配额超限 toast（20 上限）。对应 PRD v1.0 六接口与校验规则的可视化。
- **访谈记录落盘约定**（流程修订）：`write-a-prd` / `grilling` 技能新增硬性要求——访谈结束即把问答原文（问题、选项、推荐答案、用户实际答案、后续推翻修订）写入 `plans/<feature>-interview.md`，作为 `Qn` 溯源的单一事实源；`README.md` 目录结构与「决策可追溯」约定同步更新。动机：`scheduled-task-prd.md` r2 引用 Q1–Q21，但访谈原文仅存于已丢失的会话历史，`Qn` 沦为死链；已定版的三个 PRD（ai-chat Q1–Q16 / chat-conversation-management Q1–Q18 / chat-conversation-content Q1–Q5）访谈原文同样未落盘，待需要时按 PRD 引用反推重建并标注"重建"

## 2026-08-21

### Added
- **定时任务 v1 PRD 草案**（`plans/scheduled-task-prd.md`，r2）：write-a-prd 流程两轮访谈共 21 问（Q1–Q21）全部采纳推荐；用户终审提出部署形态更正为**多实例共库**（推翻 Q1 单实例前提），r2 修订——
  - 调度：各实例 `@Scheduled` 轮询 + **DB 行级原子抢占**（单语句 `UPDATE ... WHERE run_state='IDLE'`，即 `SELECT FOR UPDATE` 折叠形式），多实例同一任务至多一个执行，不引入 Quartz/xxl-job
  - 到期比较用 DB `NOW()` 消除实例时钟偏差；僵死复位带 RUNNING 守卫（多实例至多一个成功）
  - 执行：非流式 `Application.call`（超时 120s 可配）；失败不重试记 `last_run_status`；任务严格串行
  - 数据：新表 `ai_scheduled_task` + 绑定会话组 `type=SCHEDULED(6)`，产出写 user/assistant 两行（`params` 标 `trigger/taskId`）；连续记忆复用 sessionId；现有两表不改 DDL
  - 接口：`/task` 六接口（创建/分页列表/更新/删除/手动执行/详情）；cron 相邻触发 ≥5 分钟、每用户 20 个上限、prompt ≤8000
  - 顺带修正：`GET /chat/conversations` 硬编码 `type=CHAT` → type 可选过滤（对齐管理 PRD 原决策）
  - 测试四层：cron 校验 / 调度触发（固定 Clock）/ 执行服务（mock 百炼）/ 六端点 MockMvc
- **定时任务数据模型拆解**（`requirements/scheduled-task/04-数据模型.md`）：`ai_scheduled_task` 建表语句（含 `idx_sched`/`idx_user` 索引）、`last_run_time` 兼作 RUNNING 起算（省 `running_since` 列）、既有表 `type=6` 与 `params` 扩展用法、关键写路径（抢占/成功/失败/僵死复位/建删）与调度语义对应表

## 2026-08-20

### Added
- **会话内容管理（只读查询）v1 PRD 定版**（`plans/chat-conversation-content-prd.md` v1.0）：访谈 5 问（Q1–Q5）全部确认，用户终审通过。范围极小——会话内容**只读查询**；创建在 chat 接口、删除随会话组。核心决策：
  - 仅一个查询接口 `GET /chat/conversation/{conversationId}/contents?page=&size=`（与 chat-conversation PRD FR-04 为同一接口，合并开发）
  - `page` 默认 1、`size` 上限 50（非法 → `PARAM_ERROR`）、`id` 升序
  - 返回 `id/role/content/crtTime/models/inputToken/outputToken`，不含内部列
  - 归属校验：不存在/非本人/已逻辑删除 → 统一 `NOT_FOUND`
  - 测试：service 单测 + 1 条 MockMvc
  - 前置差距：`PARAM_ERROR` 未定义、分页插件未配置（与既有模块共用）
- **设计承接标注**（`designs/chat-conversation-design.md`）：新增头部关联补充 + 「§7.1 会话内容只读查询的定位」——明确历史查询接口与该 PRD 为同一实现（内容创建在 chat、删除随会话组、本模块只读、不做单条查询），不另立平行设计文档；README 索引关联至 §7.1

### Changed
- **设计文档 v1.1**（`designs/ai-chat-design.md`）：应用户要求将用户登录、身份校验从 ai-chat 设计中剥离——总览图移除 AuthUserFilter 节点（入口改为「已认证请求」）、Controller 职责去掉 JWT 捕获、线程模型改为「用户上下文快照由基础设施提供」的契约式描述、D3 改写为「认证剥离至模块边界外」；模块仅保留对用户上下文的**消费**（归属校验、落库审计字段）

### Added
- **会话组与对话内容管理 v1 PRD 定版**（`plans/chat-conversation-management-prd.md` v1.0）：一轮访谈共 18 问全部确认（Q1–Q18），覆盖 v1 Out of Scope 的「会话组列表/重命名/删除 + 历史消息查询」。核心决策——
  - 四接口：`GET /chat/conversations`、`PUT /chat/conversation/{id}/name`、`DELETE /chat/conversation/{id}`、`GET /chat/conversation/{id}/contents`
  - 列表仅元信息 `id/name/type/crt_time`（不含最近消息摘要，避免大表子查询）
  - 删除为逻辑删除（`is_del=1`），内容行保留不物理删除；已删组统一 `NOT_FOUND`
  - 分页新增 `PaginationInnerInterceptor`（当前未配置），复用 `TablePageResponse`
  - 前置差距：`ErrorCodeEnum` 补充 `PARAM_ERROR`
- **需求文档集**（`requirements/chat-conversation/`）：`01-产品概述`、`02-功能需求`（FR-01~04）、`03-接口规范`（四接口 + 时序图）、`04-数据模型`（逻辑删除语义）、`05-非功能需求与风险`（NFR-01~05、R-01~03）
- **实现 spec**（`specs/chat-conversation-spec.md`）：实现决策、测试决策（service 单测 + 4 条 MockMvc）、现状差距清单（6 项开发量）
- **设计文档**（`designs/chat-conversation-design.md`）：`ChatConversationService` 深模块/外部 seam、归属校验状态机、设计决策 D1~D8、演进预留

### Changed
- **Web 层基础能力落文档 + 封装 skill**（双轨）：
  - 需求文档 `requirements/web/01-响应包装.md` + `02-错误与异常.md`（单一事实源）：统一响应包装机制（`SuccessResponse`/`TablePageResponse`/`FailedResponse`/`@SkipWrapper`/URL exclude）、错误码 `ErrorCodeEnum` 分段与现状缺口（`PARAM_ERROR` 未定义）、异常体系（`ApiException` HTTP200 / `BaseException` 状态透传 / 全局兜底）、`ApiAssert` 断言工具
  - 实现 skill `.agents/skills/web-conventions/SKILL.md`（代码工程内）：约束 AI 编写 Controller/VO/抛异常/加错误码时的实现规则，含关键类清单、`@SkipWrapper` 适用场景、`PARAM_ERROR` 现状注意、与 auth/SSE 联动
  - README 目录与索引新增 `requirements/web/` 与「Web 层约定」行
- **接口认证抽取为单一事实源**（新增 `requirements/auth/01-接口认证.md`）：认证是全工程共识，从各模块文档中抽出独立成文（JWT / `UserContext` / 跨线程捕获-重建 / 失败错误 / 范围演进）；ai-chat 与 chat-conversation 两模块的 PRD/spec/design/需求文档中的认证描述统一改为引用 auth 文档，不再重复定义；NFR-01 更名「数据隔离」；README 目录与索引更新
- **接口定义补充**（`specs/chat-conversation-spec.md` + `designs/chat-conversation-design.md`）：spec 新增「2. 接口定义」章节（通用约定、Controller 方法签名、响应体结构、错误映射，含 `SuccessResponse`/`TablePageResponse` 具体形态），后续章节顺延编号；design 新增「2.2 接口定义细化」（Controller 方法签名、VO 字段、`RenameParam`、Service 与持久层协作）
- **设计文档结构调整**（`designs/chat-conversation-design.md`）：HTTP 接口定义从模块设计中单独拎出为独立章节「2. HTTP 接口定义」（接口清单表 + 请求响应要点 + 处理流程图）；移除 Java 代码细节，改为文字/图描述；后续章节顺延编号（模块设计→3、状态机→4、数据流→5、决策→6、测试→7、演进→8）
- **AI 对话 v1 PRD 定版**（`plans/ai-chat-sse-prd.md` v1.0）：两轮访谈共 16 问全部确认，核心决策——
  - 接口 `POST /chat/sse`（需求原文 `/sse/chat` 更正）
  - SSE 三事件协议：`message` / `error` / `end`（携带累计 token 汇总）
  - 上下文完全依赖百炼 `sessionId` 记忆
  - 断连回收 + 断连/错误轮次全落库（`params` 标记，不改 DDL）
  - `prompt` 空白/超 8000 字符、`type` 非 CHAT → 参数错误
- **文档目录重组**：新增 `README.md`（导航与约定）、`CHANGELOG.md`、`GLOSSARY.md`（术语表）
- **需求文档集**（`requirements/ai-chat/`）：`01-产品概述`、`02-功能需求`（FR-01~09）、`03-接口规范`、`04-数据模型`、`05-非功能需求与风险`（NFR-01~06、R-01~05）
- **实现 spec**（`specs/ai-chat-spec.md`）：实现决策、测试决策（seam 选择）、与现状代码的差距清单（5 项开发量）

### Superseded（已取代）
- 同日早期初版 v0.01 编号文档集（旧 README + 01~05 五篇）：内容已合并进新结构，不再单独维护
