# PRD: 模型配置管理 v1（Admin 模块）

> 状态：**v1.0（已定版）** — 2026-08-25
> 决策记录：Q1–Q34 已于 2026-08-25 Grilling 确认，详见 `requirements/model-config/grilling-decisions.md`。
> 内容已拆解至 `requirements/model-config/01~05`。

## Problem Statement

系统当前 AI 对话能力完全依赖百炼平台，模型以硬编码方式配置在 yaml 中（modelId → 百炼 Agent AppId 映射）。随着业务发展，需要支持更多第三方模型（如 OpenAI、DeepSeek）供自研 Agent 使用，但百炼模型的管理方式不可复用于第三方模型。

需要建立独立的模型配置管理基础设施：
- 第三方模型的 Provider 凭证与模型信息需要持久化存储（而非 yaml 硬编码）
- 需要管理界面供管理员运行时 CRUD，而非修改配置重启服务
- 未来 Agent / Skill / 知识库 / MCP 等能力均需要查询可用模型列表

## Solution

在 `ehub-ai-workbench` 服务中新增 `admin` 模块（同服务分包），提供第三方模型 Provider 和 Model 的配置管理能力：

- Provider 级管理：存储第三方服务的连接信息（baseUrl、API Key 加密存储、鉴权方式）
- Model 级管理：在 Provider 下挂载具体模型（模型标识符、名称、描述、排序权重）
- 管理接口：管理员 CRUD + 启用禁用 + 软删除，全量返回
- 与百炼路径**完全解耦**：百炼模型继续硬编码，不走本模块。本模块纯粹为未来自研 Agent 铺路

## User Stories

1. 作为管理员，我想在管理界面新增一个第三方 Provider（如 OpenAI），配置 baseUrl 和 API Key，以便后续 Agent 可以调用该 Provider 下的模型
2. 作为管理员，我想在 Provider 下新增具体模型（如 gpt-4o），配置名称、描述和排序权重，以便系统知道有哪些可用模型
3. 作为管理员，我想启用/禁用某个模型，以便控制模型是否可被使用
4. 作为管理员，我想调整模型的排序权重，以便控制模型在前端展示的顺序
5. 作为管理员，我想删除某个模型或 Provider，以便清理不再使用的配置
6. 作为管理员，我想在禁用/删除 Provider 时系统阻止操作（若有启用中的 Model），以便防止误操作

## Implementation Decisions

1. **模块归属（Q10/Q22）**：在 `ehub-ai-workbench` 工程中新增 `admin` 包，与 `ai-chat` 包平级，同服务部署
2. **百炼解耦（Q18/Q24/Q27）**：百炼模型完全不入库、不走本模块。百炼模型前端硬编码下拉、`POST /chat/sse` 调用链不变。`GET /chat/models` 标记 `@Deprecated` 迁移到 v2。两套路径零交集
3. **存储结构（Q6/Q11）**：新建 `ai_model_provider` + `ai_model` 两张表，Provider 级存凭证（baseUrl + apiKey），Model 级存模型身份信息
4. **Provider code（Q31）**：预定义枚举（OPENAI、DEEPSEEK、CLAUDE、CUSTOM），管理界面下拉选择
5. **必填约束（Q32）**：Provider 的 `api_key` 和 `base_url` 均必填
6. **API Key 加密（Q19）**：AES 对称加密存储，主密钥通过 env 注入
7. **Model 参数（Q14）**：v1 仅存身份信息（id、name、description、provider_model_id、weight），不存推理参数
8. **状态管理（Q28）**：Provider 和 Model 均使用 `enabled` 字段（0/1），无额外上线状态
9. **排序方式（Q23）**：Model 表 `sort_weight` 字段（int），越小越靠前
10. **级联规则（Q21）**：Provider 有启用中的 Model 时，不允许禁用/删除该 Provider
11. **删除策略（Q17）**：软删除（`is_del=1`）。Provider 删除时级联软删除其下所有 Model
12. **分页策略（Q29）**：管理接口全量返回，不分页
13. **审计（Q33）**：标准审计字段（crt_user/crt_time/upd_user/upd_time）+ 后端接口操作日志打印（info 级，含操作人、操作类型、目标 ID）
14. **权限（Q8）**：管理员权限控制，接口遵循全工程统一认证约定
15. **v1 交付范围（Q7/Q25）**：仅管理界面 + 配置存储 + CRUD 接口。不实现任何第三方模型的调用代码。对话调用等 Agent 开发完成后新增独立接口
16. **会话 Provider 绑定（Q9）**：未来第三方对话创建会话时，在 `ai_conversation.params` 中记录 `{"provider":"openai","modelId":"gpt-4o"}`。v1 不涉及
17. **模型选择灵活性（Q20）**：未来第三方对话不强制要求续聊时必须使用原模型，每次可切换。v1 不涉及
18. **GET /chat/models 处置（Q30）**：标记 `@Deprecated`，迁移到 v2。v1 保留原有行为不变

## Testing Decisions

**策略**：纯 CRUD 业务逻辑，单元测试覆盖核心校验规则。

优先覆盖：
1. **Provider CRUD**：新增、编辑、查询、软删除的正常流程
2. **Provider code 枚举校验**：不允许非预定义的 code（CUSTOM 除外）
3. **Provider API Key 加解密**：入库加密、出库解密、查询列表时脱敏展示
4. **Model CRUD**：新增、编辑、查询、软删除
5. **级联阻断**：Provider 有启用 Model 时禁用/删除被拒
6. **Model modelId 唯一性**：同一 Provider 下 provider_model_id 不可重复
7. **排序**：按 sort_weight 升序返回
8. **软删除过滤**：查询列表自动排除 `is_del=1` 的记录
9. **操作日志**：关键操作打印 info 日志

## Scope

### v1 范围内
- ai_model_provider 表 + CRUD 接口
- ai_model 表 + CRUD 接口
- API Key AES 加密存储
- 管理界面（仅后端接口定义，UI 实现不在本 PRD 范围）
- 操作审计字段 + 日志打印

### v1 范围外
- 第三方模型的实际调用代码（Provider 适配器）
- 第三方模型的对话接口
- 模型推理参数管理（temperature、maxTokens 等）
- 用户自配模型（BYOK）
- 操作日志表（before/after 快照）
- 模型测试连通性
- 与百炼模型的任何交互
