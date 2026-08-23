# Spec — 会话组与对话内容管理 v1

> 关联 PRD 母本：`plans/chat-conversation-management-prd.md` (v1.0) ｜ 关联需求：`requirements/chat-conversation/01~05` ｜ 关联设计：`designs/chat-conversation/chat-conversation-design.md` ｜ 状态：已定版
>
> 本文约束**实现**：技术决策、测试 seam、与现状代码的差距。决策可溯源至母本访谈问题号（Qxx）。

## 1. 总体策略

在 v1 已落库数据之上的**管理面补齐**：新增会话组/内容管理接口，复用现有表、归属校验、响应包装机制；**不重写对话主流程、不改 DDL**。

## 2. 接口定义

> 接口契约（字段表、请求/响应示例、错误表）以 `requirements/chat-conversation/03-接口规范.md` 为准；本节补充**实现层约束**（HTTP 细节、Controller 方法签名、统一包装行为、错误码映射）。

### 2.1 通用约定

| 项 | 值 |
|---|---|
| 前缀 | `/chat`（完整 `/ai-workbench/chat`，`server.servlet.context-path=/ai-workbench`） |
| 认证 | 全工程统一约定，见 [`requirements/auth/01-接口认证.md`](../requirements/auth/01-接口认证.md) |
| 请求 | 普通 JSON（`Content-Type: application/json`），**非 SSE** |
| 响应包装 | 统一 `ResponseBodyAdvice`；返回 `ResponseType` 实现（`TablePageResponse`/`SuccessResponse`）原样输出，不二次包装 |
| 用户标识 | 从用户上下文取 `userId`，**不接收客户端传入的用户标识**（见 auth 文档） |
| 时间格式 | `crt_time` 序列化为 ISO-8601（如 `2026-08-20T10:00:00`，Jackson 默认） |

### 2.2 接口清单与 Controller 方法签名

```java
@RestController
@RequestMapping("/chat")
public class ChatConversationController {

    /** GET /chat/conversations?page=&size=&type= → TablePageResponse<ConversationVO> */
    @GetMapping("/conversations")
    TablePageResponse<ConversationVO> pageConversations(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) Integer type);

    /** PUT /chat/conversation/{conversationId}/name，body {name} → SuccessResponse<Void> */
    @PutMapping("/conversation/{conversationId}/name")
    SuccessResponse<Void> rename(@PathVariable Long conversationId, @RequestBody RenameParam param);

    /** DELETE /chat/conversation/{conversationId} → SuccessResponse<Void> */
    @DeleteMapping("/conversation/{conversationId}")
    SuccessResponse<Void> delete(@PathVariable Long conversationId);

    /** GET /chat/conversation/{conversationId}/contents?page=&size= → TablePageResponse<ContentVO> */
    @GetMapping("/conversation/{conversationId}/contents")
    TablePageResponse<ContentVO> pageContents(
            @PathVariable Long conversationId,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size);
}
```

**请求体 `RenameParam`**：`name`(string, 必填, 非空白且 ≤100 字符)。

### 2.3 响应体结构

| 返回类型 | 结构 | 说明 |
|---|---|---|
| `TablePageResponse<ConversationVO>` | `{code:0, total:long, data:[ConversationVO]}` | 列表/历史查询；空列表 `total=0,data=[]` |
| `SuccessResponse<Void>` | `{code:0, data:null}` | 重命名/删除成功 |

**`ConversationVO`**：`id`(long) / `name`(string) / `type`(int) / `crtTime`(date)。
**`ContentVO`**：`id`(long) / `role`(string) / `content`(string) / `crtTime`(date) / `models`(string) / `inputToken`(int) / `outputToken`(int)。

### 2.4 错误映射

| 场景 | 错误码 | 触发位置 |
|---|---|---|
| `name` 空白/超 100 字符、`size` 非法值 | `PARAM_ERROR`（需补充枚举项） | `ChatConversationService`（流式外，普通 JSON 错误） |
| 会话组不存在 / 非本人 / 已逻辑删除 | `NOT_FOUND "会话不存在"` | `ChatConversationService.getOwnedConversation` |
| 认证失败（无有效 JWT） | 见 [`requirements/auth/01-接口认证.md`](../requirements/auth/01-接口认证.md) | AuthUserFilter（基础设施） |

## 3. 实现决策

1. **接口**（Q15）：独立管理 Controller（`ChatConversationController`），路径沿用 `/chat` 前缀，普通 JSON（非 SSE，走统一响应包装）：
   - `GET /chat/conversations?page=&size=&type=` → `TablePageResponse<ConversationVO>`
   - `PUT /chat/conversation/{conversationId}/name`（body `{name}`）→ 重命名
   - `DELETE /chat/conversation/{conversationId}` → 逻辑删除
   - `GET /chat/conversation/{conversationId}/contents?page=&size=` → `TablePageResponse<ContentVO>`
2. **会话组列表**（Q3/Q8/Q11）：`page/size` 分页；列表项仅 `id/name/type/crt_time`（无最近消息摘要）；`crt_time` 降序；`type` 可选过滤
3. **历史消息**（Q5/Q10/Q14）：`id` 升序；返回 `id/role/content/crt_time/models/inputToken/outputToken`；不返回 `tools/params/media_content`
4. **删除**（Q4/Q13）：置 `ai_conversation.is_del=1`；内容行保留不物理删除；已删组不可见
5. **归属校验**（Q7/Q12）：不存在/非本人/已删统一 `NOT_FOUND "会话不存在"`
6. **重命名校验**（Q6/Q18）：`name` 非空白且 ≤100 字符；允许重名
7. **分页实现**（Q9）：**前置开发项**——新增 `PaginationInnerInterceptor`（`maxLimit` 上限），使用 `Page`；复用 `TablePageResponse`
8. **分页边界**（Q11/Q18）：`page<1` 归 1；`size>50` 截断；`size` 非法 → `PARAM_ERROR`；空列表 `total=0,data=[]`
9. **错误码**：`PARAM_ERROR` / `NOT_FOUND`；`ErrorCodeEnum` 当前无 `PARAM_ERROR` 枚举项，需补充（与 v1 spec 共用）
10. **并发边界**（Q17）：删除不取消进行中的 SSE 流，内容行照常写入，已删组不现列表

## 4. 测试决策

**策略（Q16）**：service 层单测（mock mapper）为核心 + 4 条 MockMvc 集成测试（每端点 1 条）；延续 v1 的 mock 策略，不引入 Testcontainers。

### 4.1 Seam 选择

- **service seam：`ChatConversationService` 各方法**（列表/重命名/删除/历史查询）——单测 mock `AiConversationMapper`/`AiConversationContentMapper`，直接断言归属校验、分页参数、`NOT_FOUND`/`PARAM_ERROR` 行为
- **Controller 层集成测试**（MockMvc，4 条，每端点 1 条）：覆盖 HTTP 通路、路径参数、Query 分页参数、统一异常包装（`NOT_FOUND`/`PARAM_ERROR`）

### 4.2 优先覆盖的行为（外部行为导向）

| # | 行为 | 对应需求 |
|---|---|---|
| 1 | 会话组列表：仅当前用户、`type` 过滤、`crt_time` 降序、空列表 `total=0,data=[]` | FR-01 |
| 2 | 历史消息：归属校验、`id` 升序、分页、不含内部列 | FR-04 |
| 3 | 归属校验：不存在/非本人/已删统一 `NOT_FOUND`（文案一致） | FR-02/03/04 |
| 4 | 重命名校验：`name` 空白/超 100 → `PARAM_ERROR`；合法名更新成功 | FR-02 |
| 5 | 删除：置 `is_del=1`、内容行保留、已删组后续不可操作 | FR-03 |
| 6 | 分页边界：`page<1` 归 1、`size>50` 截断、`size` 非法 → `PARAM_ERROR` | FR-01/04 |

### 4.3 好测试的标准

- 只断言外部可观察行为（HTTP 状态/响应体、service 返回、mapper 调用参数），不断言内部实现细节
- 归属校验三种失败分支（不存在/非本人/已删）断言**一致**的错误响应

## 5. 与现状代码的差距（开发量）

| # | 差距 | 关联 |
|---|---|---|
| 1 | 新增 `ChatConversationController`（4 个端点） | Q15 |
| 2 | 新增 `ChatConversationService`（列表/重命名/删除/历史查询 + 归属校验） | Q15 |
| 3 | 新增 `ConversationVO` / `ContentVO` 视图对象 | Q3/Q10 |
| 4 | 新增 `PaginationInnerInterceptor`（MyBatis-Plus 分页插件，当前未配置） | Q9 |
| 5 | `ErrorCodeEnum` 补充 `PARAM_ERROR` 枚举项 | Q18 关联 |
| 6 | 补充 service 单测 + 4 条 MockMvc 集成测试 | Q16 |
