# 任务书原文

本目录保存仓库主人下发的任务书原文，按原样保留，不做修改。

| 文件 | 内容 |
|---|---|
| [TASK.md](TASK.md) | 第一轮任务书：Phase 0–8；硬性规则 R1–R14、阶段报告格式（§5）、最终验收清单（§6）、停止条件（§7） |
| [TASK_v2.md](TASK_v2.md) | 第二轮任务书：Phase 9–15；新增规则 N1–N5 |

- 两份任务书原在仓库根目录，2026-09-25（Phase 16）移入本目录，内容未改。文中"保存为仓库根目录的 `TASK_v2.md`"之类的使用说明，写的仍是当时的位置。
- 规则摘要与当前状态见根目录的 [CLAUDE.md](../../CLAUDE.md)；此后仓库主人的决定记在 [docs/verification/user-decisions.md](../verification/user-decisions.md)，各阶段的改动记在 [docs/CHANGELOG.md](../CHANGELOG.md)。
- `make check-numbers` 不扫描这两份原文（`configs/number_whitelist.yaml` 的 `skip_files`，ADR-028）：它们把禁用措辞当作规则引用，`TASK_v2.md` §2.2 还有仓库主人对论文 Table 4 的转录。论文数字一律以 `results/registry.yaml` 为准（Phase 10 已逐格核对）；那些数字是论文报告值，由官方模型在官方评测 harness 上测得，不是本仓库应用层的测量结果。
