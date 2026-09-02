# PRD: 会话内容管理（只读查询）v1

> 状态：**v1.0（已定版）** — 2026-08-20
> 决策记录：Q1–Q5 已于 2026-08-20 访谈确认（用户采纳全部推荐答案），草案经用户终审通过，定版 v1.0。
> 前置母本：`plans/ai-chat-sse-prd.md`（AI Chat v1，内容由 chat 接口落库）；`plans/chat-conversation-management-prd.md`（会话组管理，删除随会话组）。
> 定位：本 PRD **范围极小**——会话内容**只读查询**。内容创建在 chat 接口，删除随会话组，本模块仅补查询能力。

## Problem Statement

业务用户发起的对话内容（`ai_conversation_content`）已由 chat 接口完整落库，但没有接口按会话组查看这些内容。用户无法回看某次对话的角色消息、文本与 token 用量。

内容生命周期已由既有能力覆盖：**创建**在 chat 接口（每轮落库 user + assistant 两条）、**删除**随会话组（会话组逻辑删除后内容随组不可见）。因此本模块不需要任何写能力，只缺一个**查询入口**。

## Solution

在 `ehub-ai-workbench` 服务中提供**会话内容查询接口**：按会话组分页返回对话内容（角色、文本、时间、token 用量）。复用现有 `ai_conversation_content` 表与归属校验机制，不引入 DDL 变更，不提供创建/删除/修改接口。

## User Stories

1. 作为业务用户，我想按会话组分页查看历史对话内容，以便回顾和核对过往信息
2. 作为业务用户，我想让我名下的会话内容只有我自己可见，以保证数据隔离
3. 作为业务用户，我想看到每条消息的 token 用量，以便了解成本消耗

## Implementation Decisions

1. **接口形态（Q1/Q2）**：仅一个只读查询接口，路径与既有 `chat-conversation-management-prd.md` 的 FR-04 保持一致——`GET /chat/conversation/{conversationId}/contents?page=&size=` → `TablePageResponse<ContentVO>`。**不提供**单条内容查询、创建、删除、修改接口
2. **分页与排序（Q3）**：`page`（默认 1，`<1` 归 1）；`size`（默认 20，上限 50，非法 → `PARAM_ERROR`）；按 `id` 升序（对话自然顺序）
3. **返回字段（Q4）**：`id/role/content/crtTime/models/inputToken/outputToken`；**不返回**内部列 `tools/params/mediaContent`
4. **归属校验（Q5）**：查询前校验会话组归属（存在 + 本人 + 未逻辑删除），任一不满足 → 统一 `NOT_FOUND "会话不存在"`（不泄露他人会话存在性）；已逻辑删除的会话组不可查内容
5. **内容生命周期归属（用户已明确）**：创建在 chat 接口、删除随会话组——本模块**只读**，不重复实现任何写逻辑
6. **响应包装**：遵循全工程统一约定（`TablePageResponse` 免二次包装）；认证遵循归档基线 `.scratch/archive/requirements/auth/01-接口认证.md`（仍为有效约定）

## Testing Decisions

**策略**：service 层单测（mock mapper）为核心 + 1 条 MockMvc 集成测试；延续既有 mock 策略，不引入 Testcontainers。

优先覆盖的行为（外部行为导向）：

1. **归属校验**：不存在 / 非本人 / 已逻辑删除统一 `NOT_FOUND "会话不存在"`（三者文案一致）
2. **分页与排序**：`page<1` 归 1、`size>50` 截断、`size` 非法 → `PARAM_ERROR`；按 `id` 升序
3. **返回字段**：`id/role/content/crtTime/models/inputToken/outputToken`，不含内部列
4. **空数据**：无内容时返回 `total=0, data=[]`

## Out of Scope

- 会话内容的创建（由 chat 接口负责）、删除（随会话组）、修改/编辑
- 单条内容查询（按内容 id）
- 会话组列表 / 重命名 / 删除（见 `chat-conversation-management-prd.md`）
- 内容物理删除 / 归档、跨用户共享

## Further Notes

- **与既有 PRD 的关系**：`chat-conversation-management-prd.md` 的 FR-04 已定义同一接口（历史消息查询）。本 PRD 独立成文、聚焦"内容只读"这一极简范围；实现时两者为**同一接口**，不重复开发
- **现状差距**：`ErrorCodeEnum` 无 `PARAM_ERROR`（需在 2000 段补充，与既有模块共用）；MyBatis-Plus 分页插件未配置（`PaginationInnerInterceptor` 为前置开发项）
- **数据来源**：内容数据由 chat 接口每轮落库（user + assistant 两条），查询直接读 `ai_conversation_content`
- **设计承接**：接口设计与归档基线 `.scratch/archive/designs/chat-conversation/chat-conversation-design.md` 为同一实现（见其 §7.1），不另立平行设计文档
- **已定版**：本文作为**决策母本**保留；实现上与 `chat-conversation-management-prd.md` FR-04 为同一接口，合并开发
