# Grilling 决策记录 — 模型配置管理

> 日期：2026-08-25
> 模块：model-config（admin 子模块）
> 状态：已确认

## 完整决策树

| # | 决策点 | 选项 | 用户选择 | 推荐答案 |
|---|---|---|---|---|
| Q1 | 调用模式 | A)直连 B)百炼代理 C)混合 | C | C |
| Q2 | 管理方式 | A)配置级 B)运营级界面 C)用户级 | B | - |
| Q3 | 记忆策略 | A)服务端组装 B)模型原生 C)按模型分流 | C | C |
| Q4 | 向后兼容 | A)必须无缝 B)允许中断 | A | A |
| Q5 | 版本定位 | A)v1增量 B)v2大版本 | A | - |
| Q6 | 配置存储 | A)新建表 B)yaml热加载 C)Nacos | A | A |
| Q7 | v1交付范围 | A)纯基础设施 B)基础设施+直连示例 | A | A |
| Q8 | 管理权限 | A)管理员 B)角色权限 C)租户级 | A | A |
| Q9 | 会话Provider绑定 | A)存params B)反查 C)默认推断 | A | A |
| Q10 | 管理位置 | A)嵌入workbench B)独立后台 C)独立应用 | 新建admin模块 | - |
| Q11 | API Key粒度 | A)每模型独立 B)Provider+Model两级 | B | B |
| Q12 | 用户vs管理员 | A)管理员统配 B)BYOK C)混合 | A | A |
| Q13 | Provider字段 | A)最小集 B)最小集+鉴权 C)+转换配置 | B | B |
| Q14 | Model参数 | A)仅身份 B)身份+模型参数 C)+用户可覆盖 | A | A |
| Q15 | yaml迁移 | A)一次性 B)双轨并行 C)yaml兜底 | 不适用（百炼不入库） | - |
| Q16 | models响应格式 | A)保持扁平 B)+provider C)嵌套 | 不适用（百炼前端硬编码） | - |
| Q17 | 管理操作 | 确认最小CRUD+启用禁用+排序+软删除 | 按建议 | - |
| Q18 | 百炼入库 | A)存Model表 B)存Provider表 C)不入库 | C（硬编码） | - |
| Q19 | API Key加密 | A)AES对称 B)DB列级 C)KMS | A | A |
| Q20 | 非百炼模型绑定 | 不强要求与原模型一致 | - | - |
| Q21 | Provider级联 | A)级联禁用 B)阻断操作 C)独立 | B | B |
| Q22 | 管理位置 | A)同服务分包 B)独立服务 | A | A |
| Q23 | 排序方式 | A)weight B)拖拽 | A | A |
| Q24 | 百炼模型来源 | 前端硬编码，不走模型接口 | - | - |
| Q25 | 第三方对话 | 等Agent开发后新增独立接口 | - | - |
| Q26 | Admin部署 | 同服务分包 | - | A |
| Q27 | 百炼与第三方关系 | 完全解耦，两套逻辑零交集 | - | - |
| Q28 | Model状态 | 只用enabled | - | C |
| Q29 | 分页 | 全量返回 | - | A |
| Q30 | GET /chat/models | 标记迁移到v2 | - | - |
| Q31 | Provider code | 预定义枚举 | B | B |
| Q32 | Provider必填 | api_key和base_url均必填 | - | - |
| Q33 | 审计 | 标准审计字段+后端操作日志打印 | - | - |
| Q34 | UI定义范围 | 只定义后端接口和表结构 | A | A |

## 架构总结

```
v1 交付物：
├── 新表：ai_model_provider + ai_model
├── Admin 模块（同服务分包）
│   ├── Provider CRUD + 启用禁用（阻断级联）
│   ├── Model CRUD + 启用禁用 + weight 排序 + 软删除
│   └── 操作审计字段 + 日志打印
├── 百炼路径：零改动
│   ├── GET /chat/models → 标记 @Deprecated，迁移到 v2
│   └── POST /chat/sse → 保持原样
└── 第三方路径：v1 仅建基础设施，调用代码等 Agent 开发后
    ├── 新对话接口（v2+）
    └── Provider 适配器（v2+）
```
