# Work Items

`/write-a-prd`（及任何要求「create a work item」的 skill）在本仓库的发布落点。Tracker 选型：**本地 markdown**（见 `issue-tracker.md`）。

## PRD（write-a-prd）

- 产物为 markdown 文件，落 `./plans/<feature-name>-prd.md`；模板七节（Problem Statement / Solution / User Stories / Implementation Decisions / Testing Decisions / Out of Scope / Further Notes）**原样保留**
- 访谈记录随访谈结束**立即**落 `./plans/<feature-name>-interview.md`（Qn 溯源单一事实源；编号一经写出永不重排）
- **不为 PRD 创建 tracker 工单**——PRD 是本仓库的持久母本，spec/工单才进 tracker

## Spec（to-spec）

- 发布为 `.scratch/<feature-slug>/spec.md`，文件头部写 `Status: ready-for-agent`
- spec 头部与 PRD 母本互链：`PRD: plans/<feature>-prd.md (vX.Y)`；涉及 ADR 的决策互链 `docs/adr/NNNN-*.md`

## Tickets（to-tickets）

- 一工单一文件：`.scratch/<feature-slug>/issues/NN-<slug>.md`，从 `01` 编号
- 工单头部 `Status:` 行记 triage 状态（词表见 `triage-labels.md`）；阻塞关系用 `Blocked by: NN, NN` 行
