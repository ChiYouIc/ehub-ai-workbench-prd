# Spec — AI 对话 v1（POST /chat/sse）

> 关联 PRD 母本：`plans/ai-chat-sse-prd.md` (v1.0) ｜ 关联需求：`requirements/ai-chat/01~05` ｜ 关联设计：`designs/ai-chat/ai-chat-design.md` ｜ 状态：已定版
>
> 本文约束**实现**：技术决策、测试 seam、与现状代码的差距。决策可溯源至母本访谈问题号（Qxx）。

## 1. 总体策略

现有从 `ehub-tenant` 迁移的代码基本符合预期（Q1=B）：本 spec 为现状的规范化描述 + 明确的小幅调整；**不重写、不另起炉灶**。

## 2. 实现决策

1. **调用链**：Controller(SSE) → AiChatService → dashscope SDK（`Application().streamCall`）→ 百炼 Agent；`incrementalOutput=true`
2. **接口**：`POST /chat/sse`，请求 JSON、响应 `text/event-stream`、`@SkipWrapper`
3. **会话组**：隐式创建（Q3）；`conversationId` 传入则按 `crt_user` 校验归属，不存在/无归属统一 `NOT_FOUND`（Q14=A）
4. **SSE 事件协议**（Q4/Q11=B）：`message`（增量分片）/ `error`（`{code,message}`，Q6）/ `end`（`{conversationId,models,inputToken,outputToken}` 整轮累计）三事件名
5. **上下文**：完全依赖百炼 `sessionId`（Q5），首次回写不覆盖；不组装 messages、不调记忆体
6. **落库完整性**（Q12=B/Q13=B）：中断/错误轮次均落库，结局标记存 `params` 列 JSON，不改 DDL（Q8）
7. **认证**：全工程统一约定（见 [`requirements/auth/01-接口认证.md`](../requirements/auth/01-接口认证.md)）；异步线程「主线程捕获-异步线程重建」UserContext
8. **参数校验**（Q15=B/Q16=A）：`prompt` 空白或 >8000 字符、`type` 非 CHAT → `PARAM_ERROR`，流式开始前拦截，走统一异常包装
9. **断连回收**（Q9）：`onCompletion/onTimeout/onError` → dispose 百炼流订阅
10. **线程池**：沿用 `asyncThreadPool`(50,100)

## 3. 测试决策

**策略（Q10=B）**：mock 百炼 SDK 单测为核心 + 1 条 SSE 通路集成测试；真实百炼联调 dev 环境手测。

### 3.1 Seam 选择

- **首选 seam：`AiChatService.stream(ChatParam, AiChatCallback)`** —— 现有最高层接口，Controller 逻辑（emitter/线程池/UserContext 重建）在其之上。单测从 seam 以注入伪 `Flowable`（RxJava 冷流），Controller 与 SDK 均不需真实依赖
- **Controller 层集成测试**（MockMvc，1 条）：覆盖 HTTP → SSE 通路、事件帧格式、`@SkipWrapper`、参数校验错误走统一包装
- **不新增更低层 seam**（不 mock HTTP、不引入 Testcontainers）：v1 无 DDL 变更，DB 层用现有 mapper 直接验证（H2 或 dev 库，以现有测试基建为准）

### 3.2 优先覆盖的行为（外部行为导向）

| # | 行为 | 对应需求 |
|---|---|---|
| 1 | 会话路由：新建（name 取 prompt、type=CHAT）/续聊/`NOT_FOUND`（文案一致） | FR-02/03 |
| 2 | 参数校验：prompt 空白/超长、type 非 CHAT → `PARAM_ERROR` | FR-09 |
| 3 | 协议：`message` 按序、`end` 带累计汇总、SDK 抛错/空回复 → `error` + 关流 | FR-01/07 |
| 4 | 落库：user+assistant 两条、token 统计、sessionId 首次回写、中断/错误 `params` 标记 | FR-04~07 |
| 5 | 断连回收：emitter 回调后订阅被取消（dispose 标志置位） | FR-08 |

### 3.3 好测试的标准

- 只断言外部可观察行为：SSE 事件序列、落库记录内容、订阅取消标志；不断言内部方法调用次数/顺序
- 伪流构造：`Flowable.just(...)`/`Flowable.error(...)`/`Flowable.never()` 分别模拟正常/失败/挂起流

## 4. 与现状代码的差距（v1 开发量）

| # | 差距 | 关联 |
|---|---|---|
| 1 | SSE 事件名 `message`/`error`/`end`（现状：无名事件 + `complete()` 关流） | Q4/Q11 |
| 2 | 移除硬编码 `:::agent-error`，改可配置兜底 + 结构化 `error` 事件 | Q6 |
| 3 | 断连回收：注册 emitter 回调 → dispose 订阅 | Q9 |
| 4 | 中断/错误轮次补落库（`params` 标记） | Q12/Q13 |
| 5 | `prompt`/`type` 参数校验 | Q15/Q16 |

## 5. Out of Scope

同 [05-非功能需求与风险](../requirements/ai-chat/05-非功能需求与风险.md) 第 4 节。
