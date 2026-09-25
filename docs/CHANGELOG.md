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

### 本轮：TASK_v2 Phase 9–15（2026-09-24 至 2026-09-25，分支 `phase9-verification`）

本轮的目标是在真实环境中逐项验证 LIMITATIONS 中的未验证项，并补齐外部事实的核对。按仓库主人的决定，Phase 12.5 与 Phase 15 的前置修复包含一些缺陷修复与隔离加固，每项都有 ADR 与单测。Phase 15 的 GPU 步骤本轮没有执行（D22）。

| 阶段 | 变更 | 主要提交 |
|---|---|---|
| Phase 9 | 网络、CI 与分支的前置检查；保存 `TASK_v2.md`（2026-09-25 移至 `docs/process/`） | `80f8ece` |
| Phase 10 | registry 30 条对照 arXiv v3 Table 4 逐格核对，按整行录入 Base 与 AWM 两行；核实数据集与模型许可证，加入 CC-BY-4.0 署名；起草 AWM 许可证 issue | `9c10582`、`3059c35` |
| Phase 11 | 下载并接入官方数据集（revision `dde80a0`）；迷你夹具与官方 `e_commerce_33` 的接口对账；`official_data` 集成测试；按官方数据修正"空结果"判定，成功的写操作不再判为 empty（ADR-014） | `50a455d`、`d0d47a7`、`b6d0a8c`、`6abe9b6` |
| Phase 12 | 用 DeepSeek 做单次链路演示：`workbench agent run`、`awm agent`、`awm verify --mode sql` 各 1 次（N2），逐条记账 | `47abb28`、`662790a` |
| Phase 12.5 | 修复：风险分级以路由的 HTTP 方法为下限（ADR-015）；审计脱敏不再误伤时间戳（ADR-016）；按 DeepSeek 文档回传 `reasoning_content`（ADR-017）；工具定义只经原生 `tools` 传一次，预算按实测重设（ADR-018）；执行生成代码的子进程只拿白名单环境变量（ADR-019）；追加 1 次确认运行（D10） | `a535294`、`ce94caf`、`c6c9f1b`、`1d73039`、`f135189` |
| Phase 13 | serving profile 启用 `hermes` tool parser（D11，ADR-020）；`synth run` 可从 `gen task` 开始（ADR-021）；按官方价格页定价；执行 1 次真实合成（断点续跑、缓存、账本、`check_all`） | `4c88de8`、`b385891`、`02832e5`、`0e1824a` |
| Phase 14 | compose 的 vllm 参数与 profile 同步（D16）；GitHub Actions 的 `docker-smoke` 任务（原 U6 已验证）；中断时回收步骤的整棵进程树（ADR-022）；合成预算熔断（ADR-023）；Docker 镜像不得推送（D20） | `c0be5cc`、`5419a2c`、`3c3be06`、`c2a8a08`、`02c7a56` |
| Phase 15 | 前置修复：上游错误阈值与 `done_with_failures`（ADR-024）、`workbench verify`（ADR-025）、训练进程的环境白名单（ADR-026）、flash-attn 用锁定的 torch 编译（ADR-027）；新增 `workbench serve probe` 与 `scripts/redact_paste.py`；写好 `docs/runbooks/phase15-gpu.md`。GPU 步骤未执行（D22） | `ec9c017`、`93f4145`、`2198134`、`deb8780`、`0b0ec22`、`3fac63c`、`3b43467` |
| 收尾 | LIMITATIONS 分为"未验证"与"已验证"两节；README、CHANGELOG 与费用账本同步；按 `docs/process/TASK.md` §6 执行最终验收清单（`docs/verification/logs/2026-09-25-final-checklist.log`） | `1671021`、`6663871` 与记录验收清单的提交 |

费用：本轮外部 API 调用累计 ¥3.1396（上界口径，明细见 `docs/verification/cost-ledger.md`），低于 N3 的 ¥30 上限；Phase 13 之后没有再调用任何付费 API。
