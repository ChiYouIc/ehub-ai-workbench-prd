# AGENTS.md — 流程纪律（agent 在本仓库工作前必读）

本仓库按 **skill 原生约束**运行（2026-09-02 定案）：write-a-prd / to-spec / to-tickets / grill-with-docs / domain-modeling 等技能按其原生模板与产物约定工作，**本仓库不另建平行文档体系**。

## Agent skills

### Issue tracker

本地 markdown tracker：spec 与工单落 `.scratch/<feature>/`。见 `docs/agents/issue-tracker.md`。

### Triage labels

沿用五个默认标签（needs-triage / needs-info / ready-for-agent / ready-for-human / wontfix）。见 `docs/agents/triage-labels.md`。

### Domain docs

单上下文（single-context）：`CONTEXT.md` + `GLOSSARY.md` 在仓库根，ADR 落 `docs/adr/`。见 `docs/agents/domain.md`。

## 主线（skill 原生链路）

想法打磨（`grill-with-docs`，沉淀 CONTEXT/ADR）→ PRD（`write-a-prd`：`plans/` 落 PRD + interview）→ spec（`to-spec`：`.scratch/<feature>/spec.md`，ready-for-agent）→ 工单（`to-tickets`：`.scratch/<feature>/issues/NN-*.md`）→ 移交代码工程（`/implement`，内含 `/tdd` + `/code-review`）。

```mermaid
flowchart LR
    A["/grill-with-docs<br/>CONTEXT.md / docs/adr/"] --> B["/write-a-prd<br/>plans/&lt;feature&gt;-prd.md<br/>+ -interview.md（Qn 溯源）"]
    B --> C["/to-spec<br/>.scratch/&lt;feature&gt;/spec.md<br/>Status: ready-for-agent"]
    C --> D["/to-tickets<br/>.scratch/&lt;feature&gt;/issues/NN-*.md"]
    D --> E["/handoff → 代码工程<br/>/implement（/tdd + /code-review）"]
    B -.纸上谈不清时.-> P["/prototype"]
```

## 产物落点

| 产物 | 落点 | 来源 skill / 说明 |
|---|---|---|
| PRD 母本 | `plans/<feature>-prd.md` | write-a-prd（模板七节原样保留） |
| 访谈记录（Qn 溯源） | `plans/<feature>-interview.md` | write-a-prd：访谈结束立即落盘，编号永不重排 |
| ADR | `docs/adr/NNNN-*.md` | grill-with-docs / domain-modeling，按需懒创建 |
| Spec | `.scratch/<feature>/spec.md` | to-spec，头部 `Status: ready-for-agent` + PRD 互链 |
| 实现工单 | `.scratch/<feature>/issues/NN-<slug>.md` | to-tickets，`Status:` 行记 triage 状态 |
| 域模型 / 术语 | `CONTEXT.md` / `GLOSSARY.md` | domain-modeling（已存在，持续维护） |
| PRD / spec 发布落点 | `docs/agents/work-items.md` | write-a-prd 第 4 步引用 |

## 硬规则

1. **只产出 skill 原生模板约定的产物**：接口契约、数据模型等技术决策写入 PRD 的 **Implementation Decisions** 节或 **ADR**；前端无 skill 链路，视觉/交互决策同样入 PRD/ADR，细节在代码工程实现期处理。
2. PRD 版本用 `v主.次`（定版后小改动升次版本）。
3. 用语以 `GLOSSARY.md` 为准，文档间术语保持一致。
4. 需求变更：**先改 PRD / spec 再开发**（母本在 `plans/`，spec 与工单在 `.scratch/`）。
