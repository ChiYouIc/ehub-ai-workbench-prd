# Triage 标签

各 skill 以五个规范化 triage 角色说话。本文件把这些角色映射到本仓库 issue tracker 实际使用的标签字符串。

| mattpocock/skills 中的标签 | 本仓库 tracker 中的标签 | 含义                              |
| -------------------------- | ----------------------- | --------------------------------- |
| `needs-triage`             | `needs-triage`          | 待维护者评估分类                  |
| `needs-info`               | `needs-info`            | 等待提出方补充信息                |
| `ready-for-agent`          | `ready-for-agent`       | 已完全定稿，可交给 AFK agent 实现 |
| `ready-for-human`          | `ready-for-human`       | 需要人工实现                      |
| `wontfix`                  | `wontfix`               | 不予处理                          |

当 skill 提到某个角色（如「打上 AFK-ready triage 标签」）时，使用本表中对应的标签字符串。

本地 markdown tracker 中，标签字符串写在 issue 文件的 `Status:` 行上（如 `Status: ready-for-agent`）。
