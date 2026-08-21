# AI 工作台领域术语表（GLOSSARY）

本工程所有 PRD、需求文档、spec 中使用的领域术语以本表为准。定义力求紧凑：说清术语**是什么**，而不是它做什么。

## 百炼平台侧

**百炼（Bailian）**：
阿里云大模型服务平台，本系统的上游能力提供方。
_避免_：DashScope 平台、阿里 AI 平台

**Agent（智能体应用）**：
在百炼平台发布的应用编排单元（知识库+提示词+工作流）。本系统通过 `appId` 调用它，不感知其内部编排。
_避免_：机器人、助手、应用（歧义时）

**sessionId（百炼会话标识）**：
百炼 Agent 侧维持多轮上下文的会话标识，由 Agent 响应返回。本系统将其回写到会话组，作为续聊依据。
_避免_：记忆 ID（memoryId 是另一个概念）、会话组 ID

**memoryId（记忆体标识）**：
百炼平台的长期记忆体资源标识。v1 不使用（不调用记忆体接口）。
_避免_：与 sessionId 混用

**dashscope-sdk-java**：
百炼官方 Java SDK，本系统通过其 `Application.streamCall` 发起流式调用。
_避免_：手写 OkHttp+SSE（老项目旧方案，已废弃）

## 本系统领域模型

**会话组（Conversation）**：
用户视角的一次完整对话主题，对应表 `ai_conversation`，主键 `conversationId`。归属创建用户，是数据隔离与计费统计的单元。
_避免_：会话、对话、session（与百炼 sessionId 冲突）

**对话内容（Conversation Content）**：
会话组内的一条消息（user 或 assistant），对应表 `ai_conversation_content`。每轮对话产生两条。
_避免_：消息记录、聊天记录

**一轮对话（Chat Turn）**：
一次用户输入及其触发的完整 Agent 回复过程。正常/断连/错误三种结局均产生落库记录。
_避免_：一次请求、一次交互

**增量分片（Incremental Chunk）**：
`incrementalOutput=true` 时百炼每次推送的文本片段，SSE `message` 事件逐个下发，前端自行拼接。
_避免_：全量文本、流片段

**隐式创建（Implicit Creation）**：
会话组不设独立创建接口，首轮对话（不传 `conversationId`）时自动建组。与「显式创建」相对。
_避免_：自动建会话

## 协议与错误

**SSE（Server-Sent Events）**：
`text/event-stream` 单向流式推送协议。本系统唯一的前端对话通道。
_避免_：WebSocket（双向，不采用）

**SSE 事件协议（Event Protocol）**：
本系统定义的三事件协议：`message`（增量分片）/ `error`（异常，含 code）/ `end`（正常结束 + 累计 token 汇总）。
_避免_：无名事件（现状旧方案）

**断连回收（Disconnect Cleanup）**：
客户端断开后，服务端取消百炼流订阅、停止拉取，并按「中断轮次」落库的过程。
_避免_：取消请求、清理连接

**中断轮次（Interrupted Turn）**：
因客户端断连被取消的一轮对话，`params` 标记 `{"interrupted":true,...}`，assistant 落库已收到的部分文本。
_避免_：失败轮次（那是 error 轮次）

**错误轮次（Error Turn）**：
百炼调用失败或空回复的一轮对话，`params` 标记 `{"error":true,"code":...}`。
_避免_：中断轮次、异常对话

**参数错误（PARAM_ERROR）**：
流式开始前的入参校验失败（`prompt` 空白/超 8000 字符、`type` 非 CHAT），走统一异常包装返回普通 JSON，不发 SSE。
_避免_：校验异常

## 角色与流程

**业务用户（Business User）**：
持有有效 JWT 的终端用户，会话组的唯一归属者。v1 唯一认证主体。
_避免_：租户用户、会员

**UserContext**：
认证过滤器解析 JWT 后写入的 ThreadLocal 用户上下文。跨线程（异步线程池）传递需「主线程捕获-异步线程重建」。
_避免_：会话上下文、AuthUser（那是过滤器）

**捕获-重建（Capture-Rebuild）**：
主线程读取 UserContext 快照、异步线程恢复的 ThreadLocal 传递模式，SSE 长任务的标准姿势。
_避免_：ThreadLocal 透传（未明确机制的说法）
