# PRD: AI Chat 对话系统 v1（POST /chat/sse）

> 状态：**v1.2（已定版）** — 2026-08-25
> 决策记录：Q1–Q16 已于 2026-08-20 访谈确认；Q17–Q21 于 2026-08-25 补充确认；Q22–Q30（v2 预留）于 2026-08-25 确认。本文作为**决策母本**保留。
> 内容已拆解至归档基线 `.scratch/archive/requirements/ai-chat/01~05` 与 `.scratch/archive/specs/ai-chat-spec.md`（2026-09-02 体系归档）。

## Problem Statement

业务系统需要 AI 对话能力，但各业务前端不应直接对接百炼大模型平台：
- 直接对接需要每个前端管理百炼的 API Key、Agent AppId、SSE 协议细节，存在密钥泄露与重复开发成本
- 对话数据（会话组、对话内容、token 消耗）散落在百炼平台侧，公司无法留存、审计与分析
- 需要统一的用户身份体系（JWT）与对话数据权限隔离，百炼平台无法感知我们的用户

## Solution

在 `ehub-ai-workbench` 服务中对百炼平台发布的 Agent 做**二次封装**：
- 前端调用本服务的 `POST /chat/sse`（完整路径 `/ai-workbench/chat/sse`；需求原文 `/sse/chat` 更正为此），服务端解析用户身份后，将请求封装为百炼 `ApplicationParam` 调用已发布的 Agent
- 通过 `sessionId` 将同一会话组的消息在百炼侧串联（依赖百炼 Agent 的会话记忆），服务端同步保存会话组与对话内容
- 流式增量以 SSE 推送给前端；每轮对话结束后落库 user/assistant 两条消息及 token 用量

## User Stories

1. 作为业务用户，我想发起一次 AI 对话并流式看到 Agent 的回复，以便实时获取答案
2. 作为业务用户，我想在已有会话组中继续追问，以便 Agent 能结合上下文回答
3. 作为业务用户，我想让我名下的会话组/对话内容只有我自己可见，以保证数据隔离
4. 作为平台运营者，我想让每轮对话（含 token 消耗、所用模型）保存在我方数据库，以便成本核算与审计
5. 作为前端开发者，我想只对接一个稳定的 SSE 接口（无需感知百炼），以便降低接入成本
6. 作为业务用户，我想在我断开连接（关闭页面/取消）后，服务端停止继续拉取百炼输出，以便不浪费资源与 token
7. 作为业务用户，我想对 AI 回复进行评价（赞/踩），以便平台优化模型效果
8. 作为业务用户，我想重新生成上一轮 AI 回复（用不同回答替换），以便获得更满意的结果
9. 作为业务用户，我想在对话中@提及同事，以便上下文包含对方信息辅助 AI 回答
10. 作为业务用户，我想选择不同的模型进行对话，以便按场景切换模型能力
11. 作为业务用户，我想上传文件作为对话输入，以便让 AI 基于文件内容回答

## Implementation Decisions

**总体策略（Q1=B）**：现有从 `ehub-tenant` 迁移的代码基本符合预期，本 PRD 为现状的规范化描述 + 明确的小幅调整；不重写、不另起炉灶。

1. **调用链**：Controller(SSE) → AiChatService → dashscope SDK（`Application().streamCall`）→ 百炼 Agent；`incrementalOutput=true`
2. **接口路径（Q2）**：`POST /chat/sse`，`Content-Type: application/json` 请求体 `{name?, conversationId?, prompt, type?}`，响应 `text/event-stream`。需求原文 `/sse/chat` 更正为 `/chat/sse`
3. **会话组（Q3）**：隐式创建 —— `conversationId` 传入则校验归属（`crt_user` = 当前用户）后续聊，不传或无效则抛 `NOT_FOUND`；不传则新建（name 默认取首条 prompt，type 默认 CHAT=0）。v1 不提供显式「创建会话组」接口
4. **SSE 事件协议（Q4）**：新增事件名区分三类事件——
   - `event: message`：增量分片，data 为 JSON `{role, conversationId, message, models, inputToken, outputToken}`；首个分片携带 `conversationId` 供前端建立会话
   - `event: error`：异常（百炼调用失败 / 空回复兜底），data 为 `{code, message}` 结构，之后关流
   - `event: end`：正常结束标记，data 为最终汇总 `{conversationId, models, inputToken, outputToken}`（整轮累计值，修正分片累加误差），之后关流
5. **上下文记忆（Q5）**：完全依赖百炼 `sessionId`（Agent 自带会话记忆）：首次回复后从 Agent 响应中取出 `sessionId` 回写会话组；本服务不组装历史 messages、不调用百炼记忆体接口。风险：百炼侧记忆被清空则上下文丢失（见 Further Notes）
6. **错误兜底（Q6）**：去掉硬编码 `:::agent-error` 文案；Agent 异常/空回复统一走 `event: error`，兜底文案做成 `WorkbenchProperties` 可配置项（默认中文提示）
7. **认证（Q7=A）**：仅面向已登录业务用户，遵循全工程统一认证约定（见归档基线 `.scratch/archive/requirements/auth/01-接口认证.md`，仍为有效约定）。异步线程需「主线程捕获-异步线程重建」ThreadLocal 上下文（现状机制保留）。client token（服务间认证）不在 v1 范围
8. **表结构（Q8）**：沿用 ehub 库现有 `ai_conversation` / `ai_conversation_content` 两表，**不允许 DDL 变更**。现有列够用：`params`（存异常轮次 JSON 等扩展信息）、`media_content` 留空备用
9. **落库时机**：每轮流式结束后保存 user + assistant 两条 `ai_conversation_content`（assistant 记录 models/inputToken/outputToken）；新建会话组在首轮调用前先落库
10. **落库完整性（Q12=B / Q13=B）**：断连与错误轮次**均需落库**，不丢数据——
    - 客户端断连被取消：user 消息 + 已收到的部分 assistant 文本落库，`params` 存 `{"interrupted":true,"reason":"client_disconnected"}`
    - 百炼调用失败/空回复：user 消息 + assistant 错误占位（空文本或兜底文案）落库，`params` 存 `{"error":true,"code":"<错误码>"}`
11. **断连回收（Q9）**：注册 `SseEmitter.onCompletion/onTimeout/onError` 回调，客户端断连后取消百炼流订阅（`blockingForEach` 所在订阅 dispose），停止后续拉取；已产生增量的落库走第 10 条
12. **错误信息区分（Q14=A）**：`conversationId` 不存在与不属于当前用户统一报 `NOT_FOUND "会话不存在"`，不泄露他人会话存在性
13. **prompt 校验（Q15=B）**：空白或超过 8000 字符 → 参数错误（`PARAM_ERROR`）。校验发生在流式开始前，走统一异常包装返回普通 JSON 错误，而非 SSE `error` 事件
14. **type 约束（Q16=A）**：v1 只接受 CHAT=0（不传默认 CHAT；传其他值 → 参数错误），为后续多类型扩展保留枚举位
15. **线程池**：沿用 `AsyncConfig.asyncThreadPool`(50,100)，SSE 长任务与 HTTP 线程解耦
16. **模型选择（Q17=B）**：v1 支持前端选择模型。新增 `GET /chat/models` 接口返回可用模型列表（从配置读取，非百炼动态拉取）；`POST /chat/sse` 请求体新增 `modelId` 字段。服务端根据 `modelId` 映射到对应百炼 Agent AppId 调用。不传 `modelId` 则使用默认模型（配置项）。v1 模型配置为静态 yaml 列表，不提供管理界面
17. **消息评价（Q18=A）**：新增 `POST /chat/feedback` 接口，对单条 assistant 消息做赞/踩。评价存入 `ai_conversation_content.params` 字段（扩展 JSON），不改 DDL。接口校验消息归属当前用户。同一消息可重复评价（覆盖前次）
18. **重新生成（Q19=B）**：对同一 conversationId 重新调用 `POST /chat/sse`（不传 prompt，改传 `regenerateContentId` 指定要替换的 assistant 消息 ID）。服务端取该 assistant 消息对应的上一条 user 消息作为 prompt 重新调百炼；成功后新消息落库，旧 assistant 消息逻辑删除（`is_del=1`）。百炼侧 sessionId 不变（同一会话上下文保持）。错误卡重试语义等同：对 error 轮次的 assistant 消息 ID 重新生成
19. **@提及成员（Q20=B）**：新增 `GET /chat/members?keyword=` 接口，模糊搜索当前用户可见范围内的成员列表（从组织架构/通讯录服务获取，v1 首期从钉钉通讯录接口拉取并缓存）。前端输入 `@` 触发弹层调用此接口；选中成员后 prompt 中插入 `@成员名` 文本，原样传给百炼。服务端不做特殊解析——百炼 Agent 自行理解提及语义。弹层展示、过滤、键盘交互均为纯前端行为
20. **文件上传（Q21=B）**：新增 `POST /chat/upload` 接口上传文件（复用全工程文件上传基建，限制 10MB），返回 `fileId`。`POST /chat/sse` 请求体新增 `fileIds` 字段（long 数组）。服务端将文件内容读取后拼接至 prompt 前部（带文件名标记），然后调百炼。v1 仅支持文本类文件（.txt/.csv/.json/.md），非文本文件返回 `PARAM_ERROR`。上传文件暂不持久化关联到对话内容表（v2 再做）

### v2 预留决策（自研 Agent 能力）

21. **思考/工具调用过程展示（Q22=A / Q23 / Q24 / Q25 / Q27）**：自研 Agent 将支持「思考」（reasoning）和「工具调用」（tool call）中间步骤，前端需在助手消息内展示过程块。具体规则：①思考块默认折叠显示摘要 + 耗时，可展开完整内容；②工具调用块纵向堆叠按返回顺序排列；③工具入参/出参完整 JSON 展示；④工具调用失败时工具块自带错误态（tag_danger_bg + danger 文案），不阻断回复流程。v1 百炼 Agent 无此能力，仅展示最终消息文本。
22. **SSE 事件协议扩展（Q29）**：v2 新增 `event: thinking`（data: `{content, duration?}`）和 `event: tool_call`（data: `{toolName, status, input?, output?, duration?, error?}`）两种事件类型，现有 `message`/`error`/`end` 保持不变。协议细节 v2 实施前最终确认。
23. **工具调用持久化（Q26=B / Q30=B）**：v2 需新增独立表 `ai_conversation_tool_call`（DDL 扩展，违反 v1 Q8 「不允许 DDL 变更」约束但属 v2 范围），存储每次工具调用的 name/input/output/status/duration/error；assistant 消息的 `params` JSON 中只存 `toolCallIds: [id1, id2, ...]` 引用列表。不塞入 `params` 以避免 JSON 膨胀。

## Testing Decisions

**策略（Q10=B）**：mock 百炼 SDK 的单元测试为核心 + 1 条 SSE 通路集成测试；真实百炼联调用 dev 环境手测。

优先覆盖的行为（外部行为导向，不测实现细节）：
1. **会话路由**：不传 `conversationId` 新建会话组（name 默认取 prompt、type 默认 CHAT）；传入且归属正确 → 续聊；传入但不存在/不属于当前用户 → `NOT_FOUND` 错误（两者文案一致）
2. **参数校验**：`prompt` 空白或 >8000 字符 → `PARAM_ERROR`；`type` 非 CHAT → `PARAM_ERROR`
3. **流式推送协议**：分片按序产出 `message` 事件；正常结束发 `end`（携带累计 token 汇总）；SDK 抛错/空回复发 `error` 事件并关流
4. **落库**：每轮保存 user + assistant 两条记录，assistant 含 models/token 统计；`sessionId` 首次回写会话组；断连/错误轮次按 Implementation Decision 10 落库并带 `params` 标记
5. **断连回收**：emitter 完成/超时/出错后，百炼流订阅被取消（验证 dispose/取消标志被置位）
6. **模型选择**：不传 `modelId` 使用默认；传有效 `modelId` 映射到对应 Agent AppId；传无效 `modelId` → `PARAM_ERROR`
7. **消息评价**：赞/踩写入 params，重复评价覆盖前次，非本人消息 → `NOT_FOUND`
8. **重新生成**：传 `regenerateContentId` 取对应 user 消息重调百炼，旧 assistant 逻辑删除，新消息落库
9. **@提及**：`GET /chat/members` 返回成员列表，`@成员名` 原样拼入 prompt 传百炼
10. **文件上传**：`POST /chat/upload` 限制 10MB + 文本类后缀；`fileIds` 传入后文件内容拼入 prompt

测试基建：`spring-boot-starter-test`（JUnit5 + Mockito + MockMvc）已在 pom；SDK 侧通过可 mock 的 `streamCall` 入参/返回流注入伪 `Flowable`。

## Out of Scope

- ①会话组列表/重命名/删除接口 ②历史消息查询接口 ⑥输入内容审核 —— v2 再议
- 百炼记忆体显式管理（`/api/v2/apps/memory/*`）、非文本多模态输入（图片/音视频）
- client token 服务间认证
- 模型管理界面（v1 为静态配置）
- /快捷指令的后端管理（v1 纯前端预填 prompt 模板）
- 思考/工具调用过程展示（v2 自研 Agent 时实施，设计预览见归档基线 `.scratch/archive/designs/ai-chat/ai-chat-ia.md` P6；PRD 预留决策 #21~23）

## Further Notes

- 代码现状与 PRD 差异：现有路径为 `POST /chat/sse`，需求原文写的是 `/sse/chat`（已按 Q2 更正）
- Agent 异常时现有硬编码兜底文案 `:::agent-error` 将移除，改为可配置（Q6）
- 风险：上下文完全依赖百炼 `sessionId` 记忆（Q5），百炼侧记忆被清空/过期则历史上下文丢失；v2 再评估服务端记忆组装
- 风险：落库发生在异步线程池（容量 50/100），高峰期排队可能延迟写入；token 统计依赖百炼 usage 返回，缺失时记 0
- 访谈状态：Q1-Q16 于 2026-08-20 确认通过，Q17-Q21 于 2026-08-25 补充确认，Q22-Q30（v2 预留）于 2026-08-25 确认。本文档定版 v1.2。后续需求变更：先改本 PRD 再开发
- v1.1 新增能力：模型选择(Q17)、消息评价(Q18)、重新生成/重试(Q19)、@提及成员(Q20)、文件上传(Q21)
- v1.2 新增 v2 预留决策：思考/工具调用过程展示(Q22/Q23/Q24/Q25/Q27)、SSE 事件协议扩展(Q29)、工具调用持久化独立表(Q26/Q30)
