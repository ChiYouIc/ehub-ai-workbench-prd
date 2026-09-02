# Issue tracker：本地 Markdown

本仓库的 issue 与 spec 以 markdown 文件形式存放于 `.scratch/`。

## 约定

- 一个 feature 一个目录：`.scratch/<feature-slug>/`
- spec 为 `.scratch/<feature-slug>/spec.md`
- 实现 issue 一工单一文件，落 `.scratch/<feature-slug>/issues/<NN>-<slug>.md`，从 `01` 编号，**绝不**合并为单一工单文件
- Triage 状态记在 issue 文件头部的 `Status:` 行（角色词表见 `triage-labels.md`）
- 评论与对话历史追加到文件底部 `## Comments` 标题下

## 当 skill 说「发布到 issue tracker」

在 `.scratch/<feature-slug>/` 下新建文件（目录不存在则创建）。

## 当 skill 说「获取相关工单」

读取引用路径处的文件。用户通常会直接传路径或工单编号。

## Wayfinding 操作

供 `/wayfinder` 使用。**地图（map）**是一个文件，每个工单对应一个**子**文件。

- **Map**：`.scratch/<effort>/map.md`（Notes / Decisions-so-far / Fog 正文）。
- **子工单**：`.scratch/<effort>/issues/NN-<slug>.md`，从 `01` 编号，正文写问题。`Type:` 行记录工单类型（`research`/`prototype`/`grilling`/`task`）；`Status:` 行记录 `claimed`/`resolved`。
- **阻塞**：文件头部 `Blocked by: NN, NN` 行。所列文件全部 `resolved` 后该工单解除阻塞。
- **Frontier**：扫描 `.scratch/<effort>/issues/` 中开放、无阻塞、未认领的文件；编号最小者优先。
- **认领（Claim）**：动手前先设 `Status: claimed` 并保存。
- **解决（Resolve）**：在 `## Answer` 标题下追加答案，设 `Status: resolved`，然后把上下文指针（要点 + 链接）追加到 `map.md` 的 Decisions-so-far。
