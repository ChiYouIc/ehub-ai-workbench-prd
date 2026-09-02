# Domain Docs（域文档）

工程类 skill 探索代码库时，应如何消费本仓库的领域文档。

## 探索前先读这些

- 仓库根的 **`CONTEXT.md`**；或
- 仓库根的 **`CONTEXT-MAP.md`**（若存在）：它指向每个 context 一份 `CONTEXT.md`。读取与当前主题相关的每一份。
- 仓库根的 **`GLOSSARY.md`**：全工程统一术语表，本仓库的单一切片词汇源。
- **`docs/adr/`**：读取涉及你即将工作区域的 ADR。多 context 仓库中，还要检查 `src/<context>/docs/adr/` 下的 context 级决策。

以上文件若不存在，**静默继续**。不要标记其缺失；不要预先建议创建。`/domain-modeling`（经 `/grill-with-docs` 与 `/improve-codebase-architecture` 触达）会在术语或决策真正敲定时惰性创建它们。

## 文件结构

单 context 仓库（大多数仓库）：

```
/
├── CONTEXT.md
├── GLOSSARY.md
├── docs/adr/
│   ├── 0001-xxx.md
│   └── 0002-yyy.md
└── plans/
```

## 使用术语表的词汇

当你的产出提到领域概念（issue 标题、重构提案、假设、测试名），使用 `CONTEXT.md` / `GLOSSARY.md` 定义的术语。不要漂移到术语表明确回避的同义词。

如果你需要的概念还不在术语表里，那是一个信号：要么你在发明项目不用的语言（请重新考虑），要么存在真实缺口（记下来交给 `/domain-modeling`）。

## 标记 ADR 冲突

如果你的产出与既有 ADR 矛盾，显式指出，不要静默覆盖：

> _与 ADR-0007（事件溯源订单）矛盾，但值得重开，因为……_
