---
name: tapd
display_name: TAPD 项目管理查询
description: 腾讯 TAPD 命令行工具（tapd-ai-cli）。当用户提到 TAPD、需求/story、缺陷/bug、任务/task、迭代/发布评审、TAPD 评论/wiki，或给出 TAPD 链接、需求编号（如 1008539）要求查看/查询/创建/更新时，通过 tapd 命令调用 TAPD Open API 完成操作。
version: 1.0.0
author: owen
tags:
  - tapd
  - cli
  - 项目管理
---

# TAPD 项目管理查询

通过 `tapd-ai-cli` 操作腾讯 TAPD（Open API 封装），查询/创建/更新需求、缺陷、任务、迭代、Wiki 等。

## 环境与调用方式

- **安装**：从 https://github.com/studyzy/tapd-ai-cli 下载对应平台的可执行文件（Windows 用 `tapd_<ver>_windows_amd64.zip`，注意 release 文件名用 `1.0.0` 而非 `v1.0.0`），解压后将 `tapd.exe` 所在目录加入系统环境变量 PATH，之后直接以 `tapd <子命令>` 调用
- **凭据**：首次使用前执行 `tapd auth login` 完成登录；若后续报 `authentication_required`，提示用户在终端亲自执行 `tapd auth login`（凭据不要经过 AI）

## 常用命令

| 用途 | 命令 |
|------|------|
| 查看参与的项目 | `tapd workspace list` |
| 查询需求详情 | `tapd story show <id> --workspace-id <wid>` |
| 需求列表/创建/更新 | `tapd story list` / `create` / `update <id>` |
| 缺陷 | `tapd bug list` / `show <id>` / `create` / `update` |
| 任务 | `tapd task list` / `show <id>` / `todo` |
| 迭代 | `tapd iteration list` |
| 通过 URL 查任意条目 | `tapd url <tapd链接>`（需求/缺陷/任务/Wiki 通吃） |
| 评论 | `tapd comment list --entry-type story --entry-id <id>` / `add` |
| Wiki | `tapd wiki list` / `show <id>` |
| 全部命令 | `tapd --help` |

## ID 约定

- TAPD 完整 ID 形如 `1136062570001008539`（含工作区前缀），短 ID `1008539` 可直接用于 `show`/`update`
- 用户只给短 ID 时，先执行 `tapd workspace list` 确认可用工作区；无法确定时，向用户询问 `--workspace-id` 后再查询
- `story show` 默认输出 Markdown（含描述+评论），列表命令输出紧凑 JSON

## 高级过滤（--filter）

所有 `list` 命令支持可重复的 `--filter`，透传 TAPD OpenAPI 查询语法：

```powershell
# 名称模糊
tapd story list --filter "name=LIKE<已退款>"
# 时间范围
tapd bug list --filter "created=>2026-01-01" --filter "created=<2026-12-31"
# 组合状态
tapd task list --filter "status=CONTAINS_OR<开发中|测试中>"
```

操作符：`LIKE`（模糊）、`EQ` / `NOT_EQ`、`LIKE_OR`、`CONTAINS` / `CONTAINS_OR`、`USER_OR`（多人）、`>` `<` `~`（时间）、`|`（多值 OR）。适用于标准字段与自定义字段（`custom_field_*`）。

## 注意事项

1. 输出默认紧凑 JSON 省 token；仅人类阅读时加 `--pretty`。
2. 需求/缺陷详情默认 Markdown 且附带评论；加 `--no-comments` 可省略。
3. 仓库地址：https://github.com/studyzy/tapd-ai-cli
