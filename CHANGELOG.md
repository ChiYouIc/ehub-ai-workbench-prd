# 变更日志（CHANGELOG）

本文件记录文档工程的全部重要变更，按日期倒序。格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

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
