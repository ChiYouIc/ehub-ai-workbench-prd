# PRD: 会话组与对话内容管理 v1

> 状态：**v1.0（已定版）** — 2026-08-20
> 决策记录：Q1–Q18 已于 2026-08-20 访谈确认（用户采纳全部推荐答案），草案经用户终审通过，定版 v1.0。
> 前置母本：`plans/ai-chat-sse-prd.md`（AI Chat v1，会话组/内容数据已落库）。
> 本 PRD 覆盖 v1 Out of Scope 的「会话组列表/重命名/删除 + 历史消息查询」两项。

## Problem Statement

业务用户使用 `POST /chat/sse` 发起对话后，没有入口管理自己积累的会话：

- 会话组不断累积，无法回到过去的主题继续查看或使用
- 无法重命名一个"首轮 prompt 即组名"的会话组，时间一长难以辨认
- 无法删除不再需要的会话组，列表只能无限膨胀
- 没有接口按会话组回看历史对话内容（角色、文本、token 用量）

当前服务层虽有雏形（按用户列会话组 `selectChatByUserId`、按会话组查内容 `selectByConversationId` limit 100），但无任何对外管理接口，前端无法触达这些能力。

## Solution

在 `ehub-ai-workbench` 服务中新增**会话组与对话内容管理接口**（独立 Controller，统一 `/chat` 前缀），对业务用户暴露四类操作：

- **会话组列表**：分页查看当前用户全部会话组元信息，可选按 `type` 过滤
- **会话组重命名**：修改会话组名称
- **会话组删除**：逻辑删除（`is_del=1`），其下对话内容保留但随删除不可见
- **历史消息查询**：按会话组分页回看对话内容（含角色、文本、token 用量）

所有接口认证遵循全工程统一约定（见归档基线 `.scratch/archive/requirements/auth/01-接口认证.md`，仍为有效约定），并沿用 v1 数据隔离机制（`crt_user` 归属校验），复用现有 `ai_conversation` / `ai_conversation_content` 两表，**不引入 DDL 变更**。

## User Stories

1. 作为业务用户，我想分页查看我名下的会话组列表，以便找到并回到过去的对话主题
2. 作为业务用户，我想重命名某个会话组，以便日后更容易辨认和查找
3. 作为业务用户，我想删除不再需要的会话组，以便清理我的会话列表
4. 作为业务用户，我想按会话组查看历史对话内容（含 token 用量），以便回顾和核对过往信息
5. 作为业务用户，我想让我名下的会话组/对话内容只有我自己可见，以保证数据隔离

## Implementation Decisions

**总体策略**：本 PRD 是 v1 已落库数据之上的**管理面补齐**，复用现有表与归属校验机制，不重写对话主流程。

1. **接口形态（Q15）**：新建独立管理 Controller，路径沿用 `/chat` 前缀（与 SSE 接口同域，方便前端统一代理与鉴权）：
   - `GET /chat/conversations?page=&size=&type=` → 会话组分页列表
   - `PUT /chat/conversation/{conversationId}/name`（body `{name}`）→ 重命名
   - `DELETE /chat/conversation/{conversationId}` → 删除（逻辑删除）
   - `GET /chat/conversation/{conversationId}/contents?page=&size=` → 历史消息分页
2. **会话组列表（Q3/Q8/Q11）**：分页 `page/size`；列表项仅返回元信息 `id/name/type/crt_time`，**不含最近一条消息摘要**（避免大表子查询，历史内容经详情接口按需加载）；按 `crt_time` 降序；`type` 过滤可选，不传返回该用户全部类型
3. **历史消息查询（Q5/Q10/Q14）**：分页 `page/size`；按 `id` 升序（对话自然顺序）；返回 `id/role/content/crt_time/models/inputToken/outputToken`；不返回内部列（`tools/params/media_content`）
4. **删除语义（Q4/Q13）**：删除仅置 `ai_conversation.is_del=1`（逻辑删除，`@TableLogic`）；**对话内容行保留不物理删除**（保留审计与成本核算数据）；历史消息查询经会话组归属校验，已删组不可查
5. **归属校验（Q7/Q12）**：所有管理接口对「不存在 / 不属于当前用户 / 已逻辑删除」的会话组统一返回 `NOT_FOUND "会话不存在"`，不泄露他人会话存在性（延续 v1 Q14）
6. **重命名校验（Q6/Q18）**：`name` 非空白且 ≤100 字符，非法 → `PARAM_ERROR`；允许重名（无唯一约束）
7. **分页实现（Q9）**：项目当前未配置 MyBatis-Plus 分页插件，需新增 `PaginationInnerInterceptor`（设 `maxLimit` 上限），使用 `Page` 分页；分页响应复用既有 `TablePageResponse`（`{code,total,data}`）形状
8. **分页参数边界（Q11/Q18）**：`page<1` 默认 1；`size>50` 截断为 50；`size` 非法值 → `PARAM_ERROR`；空列表返回 `total=0, data=[]` 而非错误
9. **错误码（Q18 关联）**：校验失败用 `PARAM_ERROR`、缺失/无归属用 `NOT_FOUND`，与 v1 spec 约定一致；`ErrorCodeEnum` 当前尚无 `PARAM_ERROR` 枚举项，需随本功能（或 v1 开发）补充
10. **并发边界（Q17）**：删除会话组时不取消进行中的 SSE 流——流照常写内容行（内容行永不物理删除）；已删组不再出现在列表。该边界作为已知限制接受
11. **认证（Q7 关联）**：遵循全工程统一认证约定（见归档基线 `.scratch/archive/requirements/auth/01-接口认证.md`）；管理接口为短任务，无需跨线程传递，不存在 v1 的捕获-重建问题

## Testing Decisions

**策略（Q16）**：service 层单测（mock mapper）为核心 + 4 条 MockMvc 集成测试（每个端点 1 条）；延续 v1 的 mock 策略，不引入 Testcontainers。

优先覆盖的行为（外部行为导向，不测实现细节）：

1. **会话组列表**：仅返回当前用户数据；`type` 过滤生效；按 `crt_time` 降序；空列表返回 `total=0, data=[]`
2. **历史消息查询**：仅返回归属会话组内容；按 `id` 升序；分页生效；不含内部列
3. **归属校验**：不存在 / 非本人 / 已逻辑删除统一 `NOT_FOUND "会话不存在"`（三者文案一致）
4. **重命名校验**：`name` 空白或 >100 字符 → `PARAM_ERROR`；合法名更新成功
5. **删除**：置 `is_del=1`（逻辑删除）；内容行保留；已删组不可再查询/重命名/删除
6. **分页边界**：`page<1` 归 1；`size>50` 截断；`size` 非法 → `PARAM_ERROR`

测试基建：`spring-boot-starter-test`（JUnit5 + Mockito + MockMvc）已在 pom；现有测试仅 `contextLoads`，本功能补齐 service 与 web 层用例。

## Out of Scope

- 会话组置顶、搜索、消息已读、消息编辑/撤回（Q1 未纳入）
- 会话组列表中的「最近一条消息摘要」展示（Q8 明确不取，避免大表子查询）
- 删除会话组时取消进行中的 SSE 流（Q17 接受边界，仅资源回收语义，不续传）
- 对话内容行物理删除（保留审计与成本核算数据）
- 多 Agent/skill 分流、输入内容审核、文件/多模态输入（延续 v1 Out of Scope）
- 跨用户会话可见性/共享/协作者功能

## Further Notes

- **现状差距**：`ErrorCodeEnum` 无 `PARAM_ERROR` 枚举项（v1 spec 已引用）；MyBatis-Plus 分页插件未配置——两者为本功能前置开发项
- **与 v1 的关系**：本 PRD 消费 v1 已落库数据（`ai_conversation` / `ai_conversation_content`），不依赖 v1 开发完成即可先行定义；v1 未开发前，管理接口测试可用当前 service/mapper 能力直接验证
- **并发边界记录**：删除会话组与进行中 SSE 流并发的行为（Q17）已明确接受，若后续要求「删除即停流」需评估取消机制，另行评估
- **已拆解（归档基线）**：需求文档 `.scratch/archive/requirements/chat-conversation/01~05`、实现 spec `.scratch/archive/specs/chat-conversation-spec.md`、设计 `.scratch/archive/designs/chat-conversation/chat-conversation-design.md`，本文作为**决策母本**保留
