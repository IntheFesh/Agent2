# CHANGELOG

## Prompts（`src/workbench/agent/prompts/*.md`）

每次修改 prompt，都要同时提升文件头里的版本号，并在这里记录。

| Prompt | 版本 | 日期 | 变更 |
|---|---|---|---|
| plan | 1 | 2026-09-24 | 初版：只允许使用运行时注入的工具；标记会改数据的步骤；输出 1–10 步的 JSON |
| act | 1 | 2026-09-24 | 初版：每轮最多调用一个工具；被拒绝后不重试；`empty` 不算错误；根据 `error` 里的 hints 修正参数 |
| act | 2 | 2026-09-24 | 工具的描述与参数 schema 只经请求的原生 `tools` 参数传入，system prompt 里只列工具名与风险级别；加一句说明（ADR-018） |
| verify | 1 | 2026-09-24 | 初版：输出 `complete` / `missing`，以及带来源标记的 `memories` |

## Code

按阶段划分的变更记录见 `git log --oneline`（Conventional Commits）。
