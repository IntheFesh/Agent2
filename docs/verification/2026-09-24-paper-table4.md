# 2026-09-24 论文 Table 4 核对（arXiv 2602.10090 v3）

> 论文报告值，由官方模型在官方评测 harness 上测得，不是本仓库应用层的测量结果。

## 来源

| 项目 | 值 |
|---|---|
| 摘要页 | `https://arxiv.org/abs/2602.10090`（最新版本即 v3），`https://arxiv.org/abs/2602.10090v3` |
| PDF | `https://arxiv.org/pdf/2602.10090v3`（43 页，Table 4 位于第 7 页），sha256 `419b30a8f19edc92cf268d194923cf5f29b755c161e6f22672b5bfa05b5f84d5` |
| HTML | `https://arxiv.org/html/2602.10090v3`（第 4 个 `ltx_table`），sha256 `ae2316ad884dbfc785359a1234736df8a3cc6fb1a943406224cfb643bbb96b1a` |
| 访问时间 | 2026-09-24 13:17 UTC（curl；WebFetch 仍被拦截） |
| 核对人 | claude-code |

## 版本与发表状态

摘要页 Submission history 与 Comments 原文摘录：

> [v1] Tue, 10 Feb 2026 18:55:41 UTC (6,435 KB)
> [v2] Wed, 11 Feb 2026 18:20:25 UTC (6,435 KB)
> [v3] Fri, 22 May 2026 21:39:46 UTC (6,449 KB)
> Comments: Accepted to ICML 2026

与 TASK_v2 §2.1 一致。

## Table 4 标题原文（v3）

> Table 4: Results across BFCLv3, τ2-bench, and MCP-Universe benchmarks. Best performance is in bold. For BFCLv3, "Hall." refers to the hallucination category. For τ2-bench, Pass@k denotes task success rate allowing k attempts, with Pass@1 averaged over 4 runs. For MCP-Universe, we report task success rates. Note that EnvScaler did not release the 14B model.

§5.1 关于 MCP-Universe 的排除项（PDF 第 6 页）：

> We exclude 3D design tasks that require GUI and repository management tasks that require authenticated access to GitHub or Notion.

## 核对方法

1. 从 HTML 的 Table 4 解析出全部行（Base、Simulator、EnvScaler、AWM × 4B / 8B / 14B × 三个基准）。
2. 用 pypdf 提取 PDF 第 7 页文本，按行提取所有两位小数的数字（PDF 文本中粗体单元格之间缺空格，因此按数字 token 比较，而不是按整串比较）。
3. 结果：
   - HTML 与 PDF 的 Base / AWM 共 18 行、96 个单元格逐一相同；
   - 与 TASK_v2 §2.2 的转录逐一比较，96 个单元格 0 处不一致（没有触发"与 §2.2 不一致"的停止条件）。
4. 输出与命令见 `docs/verification/logs/2026-09-24-table4-compare.log`。

## 登记到 registry 的格子

按 TASK_v2 Phase 10 第 2 条，按整行录入 Base 与 AWM 两行（4B / 8B / 14B），列为 BFCLv3 的 Overall 与 Hall.、τ²-bench 的 Pass@1 与 Pass@4、MCP-Universe 的 Overall。其中包括 AWM 低于 Base 的格子（例如 4B 与 8B 的 BFCLv3 Hall.）。Simulator 与 EnvScaler 两行不录入。

| 行 | BFCLv3 Hall. | BFCLv3 Overall | τ² Pass@1 | τ² Pass@4 | MCP-Universe Overall |
|---|---|---|---|---|---|
| 4B / Base | 73.93 | 54.92 | 15.83 | 34.89 | 6.15 |
| 4B / AWM | 73.70 | 64.50 | 22.57 | 43.89 | 6.70 |
| 8B / Base | 76.42 | 53.83 | 26.44 | 50.72 | 6.70 |
| 8B / AWM | 70.80 | 65.94 | 33.45 | 55.40 | 11.17 |
| 14B / Base | 77.94 | 61.25 | 36.69 | 55.40 | 8.38 |
| 14B / AWM | 78.37 | 70.18 | 39.03 | 57.19 | 12.29 |

单位：BFCLv3 为 leaderboard 分数；τ²-bench 为通过率（%）；MCP-Universe 为任务成功率（%）。

## 结论

registry 中原有的 9 条全部与 v3 Table 4 一致，已改为 `verified: true`；新增 21 条，共 30 条。每条的 `source` 填写 `version: v3`、`table: 4`、`page: 7`、`row`、`column`。

## 附：§5.1 训练设置（只作为事实记录，不是可复现的配方）

PDF 第 6 页 §5.1 与第 18 页 Table 12，与 TASK_v2 §2.4 一致：

> we train agents on a subset of AWM consisting of 526 environments and 3,315 tasks. Each model is trained for up to 96 optimization steps with a fixed learning rate of 7×10−7. We use a batch size of 64 and 16 rollouts … The sliding window size and maximum interaction turn are set to w=3 and 20

合成与裁判模型为 GPT-5（"The AWM pipeline is instantiated using GPT-5 … which is also used as the code-augmented LLM-as-a-Judge"）。本仓库不据此创建任何训练配方（ADR-012 不变）。
