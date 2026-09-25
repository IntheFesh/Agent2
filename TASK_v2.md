# Claude Code 补完任务书 v2：BizAgent Workbench 收尾（Phase 9–15）

> 使用方式：把本文件保存为仓库根目录的 `TASK_v2.md`，然后发送："完整阅读 TASK.md、CLAUDE.md 和 TASK_v2.md，严格按 TASK_v2.md 执行，从 Phase 9 开始。"
> **必须在能访问 huggingface.co 和 arxiv.org 的机器上运行**（例如你自己的电脑）。上一轮失败的项目，大部分就是因为沙箱拦截了这两个域名。
> Phase 15 需要 GPU，可以放到 AutoDL 上单独跑，也可以直接跳过。

---

## 0. 背景与总目标

仓库 `IntheFesh/Agent2` 当前分支为 `claude/kind-gauss-3clgyp`，HEAD 是 `f140123`，Phase 0–8 已经完成。已由第三方复核通过的有：

- `make setup / lint / test / check-numbers / doctor` 全部通过（177 passed，mypy --strict 无问题）；
- src、scripts、Makefile 中没有任何评测调用；
- 两个 submodule 固定在 `85e322f`（AWM）和 `1256586`（AgentFly），工作区干净。

本轮目标只有一个：**把 `docs/LIMITATIONS.md` 中因环境受限而未验证的项目，在真实环境里逐项验证或如实保留**，同时补齐外部事实的核对（论文数字、许可证）。**本轮不新增任何功能。**

---

## 1. 规则

**TASK.md 的 R1–R14 全部继续有效。** 尤其是以下几条：

- R1：不跑评测，不做批量统计；
- R2：不改官方数据和配置，只允许 smoke 训练；
- R3：性能数字只能来自 registry；
- R8：诚实汇报；
- R12：每个阶段结束后停下等我回复"继续"。

本轮新增以下规则：

**N1 证据留档。**
- 每一项外部核实（论文表格、许可证、数据集文件数、模型卡），都在 `docs/verification/<YYYY-MM-DD>-<topic>.md` 里记录：URL、访问时间、原文摘录（不超过 5 行）、结论。
- 每一项"在真实环境中跑通"的验证，都保存命令和输出结尾，放在 `docs/verification/logs/` 下。日志中的 key、token、邮箱、手机号必须先脱敏。

**N2 单次演示。**
- 需要真实 LLM 的链路演示（`workbench agent run`、`awm agent`、`awm verify`），每种命令最多执行 1 次，只用 1 个任务。
- 输出只能作为"链路打通"的证据，不得汇总成任何比率，也不得写成"效果"。

**N3 费用上限。**
- 本轮所有外部 API 调用的总花费上限为 **¥30**（我可以在这里修改）。
- 每次调用前先估算费用并累计记账；预计会超出上限时，停下来问我。

**N4 分支与历史。**
- 在当前 HEAD 上新建分支 `phase9-verification`，本轮所有工作都在这个分支上完成。
- 不得改写历史，不得 force push。新提交使用本机的 git 配置。

**N5 我的授权**（我勾选 `[x]` 的才可以执行，没勾选的一律不做）：
- [ ] 用 `claude/kind-gauss-3clgyp` 的当前 HEAD 创建 `main` 分支，并通过 `gh` 把它设为仓库默认分支
- [ ] 本轮结束后，从 `phase9-verification` 向 `main` 开 PR
- [ ] 在 Snowflake-Labs/agent-world-model 上开一个 issue，礼貌地询问许可证。只能用我审核过的措辞：先把草稿写进 `docs/verification/awm-license-issue-draft.md`，等我确认后再发

---

## 2. 我已经在外部核对过的信息（仅供交叉核对，以你自己读到的原文为准）

### 2.1 论文版本
- arXiv 2602.10090 的最新版本是 **v3（2026-05-22）**，注释为 "Accepted to ICML 2026"。
- v1 发布于 2026-02-10，v2 发布于 2026-02-11。

### 2.2 v3 Table 4 中 Base 行与 AWM 行的数值（我的转录）

BFCLv3，列依次为 Non-Live / Live / Multi-Turn / Hall. / Overall：

| 尺寸 | 行 | Non-Live | Live | Multi-Turn | Hall. | Overall |
|---|---|---|---|---|---|---|
| 4B | Base | 61.44 | 63.95 | 39.38 | 73.93 | 54.92 |
| 4B | AWM | 78.60 | 76.91 | 37.99 | 73.70 | 64.50 |
| 8B | Base | 59.58 | 58.03 | 43.88 | 76.42 | 53.83 |
| 8B | AWM | 80.44 | 72.39 | 45.00 | 70.80 | 65.94 |
| 14B | Base | 69.48 | 67.28 | 47.00 | 77.94 | 61.25 |
| 14B | AWM | 81.46 | 77.20 | 51.88 | 78.37 | 70.18 |

τ²-bench，列依次为 Airline / Retail / Telecom / Pass@1 / Pass@4：

| 尺寸 | 行 | Airline | Retail | Telecom | Pass@1 | Pass@4 |
|---|---|---|---|---|---|---|
| 4B | Base | 21.00 | 19.96 | 9.43 | 15.83 | 34.89 |
| 4B | AWM | 19.00 | 30.26 | 16.45 | 22.57 | 43.89 |
| 8B | Base | 26.50 | 34.43 | 18.42 | 26.44 | 50.72 |
| 8B | AWM | 38.50 | 41.23 | 23.47 | 33.45 | 55.40 |
| 14B | Base | 27.00 | 65.35 | 12.28 | 36.69 | 55.40 |
| 14B | AWM | 31.50 | 63.60 | 17.76 | 39.03 | 57.19 |

MCP-Universe，列依次为 Location / Financial / Browser / Web / Multi / Overall：

| 尺寸 | 行 | Location | Financial | Browser | Web | Multi | Overall |
|---|---|---|---|---|---|---|---|
| 4B | Base | 2.86 | 25.00 | 0.00 | 0.00 | 0.00 | 6.15 |
| 4B | AWM | 0.00 | 22.50 | 2.94 | 2.00 | 5.00 | 6.70 |
| 8B | Base | 0.00 | 22.50 | 5.88 | 2.00 | 0.00 | 6.70 |
| 8B | AWM | 8.57 | 35.00 | 8.82 | 0.00 | 5.00 | 11.17 |
| 14B | Base | 0.00 | 27.50 | 5.88 | 2.00 | 5.00 | 8.38 |
| 14B | AWM | 8.57 | 32.50 | 11.77 | 4.00 | 0.00 | 12.29 |

表注要点：τ² 的 Pass@1 是 4 次运行的平均值；MCP-Universe 报告的是任务成功率，并排除了需要 GUI 的 3D 设计任务和需要登录鉴权的仓库管理任务。

### 2.3 许可证
- **AgentWorldModel-1K 数据集**：页面标注 **cc-by-4.0**。
- **Arctic-AWM-4B**：页面标注 **apache-2.0**，模型树显示基座为 Qwen/Qwen3-4B。
- **8B、14B**：我没有核对。
- **AWM 代码仓库**：截至 HEAD `85e322f`，仍然没有 LICENSE。

### 2.4 训练设置（论文 §5.1，仅作为事实，不是可复现的配方）
- 合成与裁判用的是 GPT-5；
- 在 526 个环境、3,315 个任务上训练；
- 最多 96 个优化步，学习率 7e-7；
- batch 64 × 16 个 rollout；
- 滑动窗口 w=3，最大交互轮数 20。

---

## 3. 分阶段任务

### Phase 9 — 环境前置检查与收尾

1. **网络检查**：确认本机能访问 `huggingface.co`、`arxiv.org`、`github.com`，能通过 `hf` 或 `huggingface-cli` 匿名下载公开数据集。任一项不通就停下。
2. **CI 状态**：用 `gh run list` 确认 `f140123` 触发的 CI 结果，写入 `docs/verification/`。如果失败，只修复 CI 问题，不改功能。
3. **建分支**：按 N4 创建 `phase9-verification`。
4. 按 N5 中我的授权处理 `main` 分支与默认分支设置；没有授权就跳过，并在报告中提醒我。
5. 列出本轮计划触碰的文件，以及每项的预计 API 花费（N3）。

**验收**：`docs/verification/2026-xx-xx-preflight.md` 记录网络、CI 与分支状态。

### Phase 10 — 外部事实核对（只改文档与 registry）

1. **论文数字**：
   - 打开 arXiv 2602.10090 **v3** 的 PDF 或 HTML，逐格核对 registry 中现有的 9 条。
   - 填写 `version: v3`、`table: 4`、`row`、`column`、`unit`（BFCL 是分数，τ² 是通过率 %，MCP-Universe 是成功率 %）。
   - 把 `verified` 改为 `true`，并新增 `verified_by: claude-code`、`verified_at: <date>`、`evidence: docs/verification/<file>`。
2. **扩充 registry**：
   - 按**整行**录入以下列的 Base 行和 AWM 行（4B / 8B / 14B 全部）：BFCLv3 的 Overall 与 Hall.、τ² 的 Pass@1 与 Pass@4、MCP-Universe 的 Overall。
   - **不得只挑有利的格子。** 例如 4B 在 τ² 上 AWM 不是最优、在 BFCL Hall. 上低于 Base，这些也必须如实录入。
   - Simulator 与 EnvScaler 两行不录入。
3. **与 §2.2 交叉核对**：任何一格与 §2.2 不一致，先停下报告，由我决定以哪个为准。你自己读到的原文优先，不得"两边折中"。
4. **模型身份**：逐个打开 Arctic-AWM-4B / 8B / 14B 的模型卡，记录许可证、基座模型、chat template 要点、上下文长度。在 registry 的 `models` 段把证据写进 `presumed_identity`。
   - 模型卡是否明确说明"这就是论文 Table 4 中的 AWM 行"，要如实写。没有明说就保持 `presumed`，不得改成已确认。
5. **许可证**：
   - 更新 UPSTREAM.md §3：数据集 CC-BY-4.0，以及三个模型各自的许可证。
   - 重新检查 AWM 仓库 HEAD 是否已有 LICENSE、PR #17 的状态。
   - 由于数据集是 CC-BY-4.0，要求署名：在 README 与 UPSTREAM.md 中加入规范的署名段落（作者、标题、链接、许可证名，以及"是否有改动"的说明）。
6. **重新生成与同步**：
   - 运行 `make results` 重新生成 RESULTS.md，并加上 §2.2 提到的表注要点；
   - 运行 `make check-numbers`，必须通过；
   - README 中"全部标注为待核对"的表述要同步更新；
   - LIMITATIONS.md §2 按实际状态更新。

**验收**：
- registry 中每条都是 `verified: true`，且带 evidence 文件；
- RESULTS.md 与 registry 一致；
- `make lint test check-numbers` 全部通过。

### Phase 11 — 接入官方数据集（CPU）

1. 运行 `make data`，把 AgentWorldModel-1K 下载到 `data/awm1k/`（不入库）。
2. **校验文件**：校验 8 个文件是否都存在，条目数是否与数据集卡一致（场景类文件各 1,000 条，verifier 类各约 10K 条）。
   - 这些是数据集事实，不是性能数字。如果被 number guard 拦截，按 `configs/number_whitelist.yaml` 的规则登记，并写明理由和来源。
3. **目录与启动**：
   - 运行 `workbench env search`，确认官方场景目录可用；
   - 通过 env manager 真实启动官方场景 `e_commerce_33` 并调用 `list_tools`；
   - 把工具清单保存到 `docs/verification/`。
4. **夹具对账**：
   - 把 `tests/fixtures/awm_mini` 中暂定的工具名，与官方 `e_commerce_33` 的 `list_tools` 结果逐一对比，输出对照表（一致 / 不一致 / 仅夹具有 / 仅官方有）；
   - 按对照表修正夹具，并更新 MANIFEST 与 LIMITATIONS 中"工具名暂定"的说明；
   - 夹具中凡是取自官方数据的内容，都要注明来源并遵守 CC-BY-4.0 署名；夹具仍然只包含最小必要内容，不得整段复制官方数据。
5. **新增集成测试**：加入 `@pytest.mark.official_data`，本地无数据时自动 skip，保证 CI 不依赖官方数据；本地有数据时必须通过。
6. `workbench doctor` 中 dataset 一项应变为 ok。

**验收**：
- 带数据时 `make test` 与 `pytest -m official_data` 都通过；
- 不带数据时（CI）`make test` 通过。

### Phase 12 — 真实 LLM 链路演示（CPU + API，受 N2、N3 约束）

1. **配置后端**：用 `openai_compat` 后端接入一个 OpenAI 兼容端点（默认 DeepSeek，key 只从环境变量读取）。运行 `workbench doctor`，llm 一项应为 ok。
2. **本仓库链路**：对官方 `e_commerce_33` 中**1 个**需要写操作的任务，执行一次 `workbench agent run`，走完"规划 → 调用 → 审批 → 完成"，并保存 trace 与 DB diff。
3. **上游链路**：
   - `awm agent`：只执行 1 次，模型端点用同一个 OpenAI 兼容服务；
   - `awm verify --mode sql`：只执行 1 次；
   - 两者的参数都以 RECON 中核实过的签名为准；
   - 用 UI 的轨迹查看器打开 `awm agent` 的输出，确认格式兼容。
4. **兼容性问题**：如果真实模型的工具调用格式与本仓库的解析不兼容，只能在 `src/workbench/llm/` 中修复，补单测，并写 ADR。不得修改上游，不得为迁就模型而放宽网关策略。
5. **文档**：WALKTHROUGH 中对应步骤改为本次真实输出，标注"单次链路演示，不构成评测"；LIMITATIONS 中的 U2、U8 按实际结果更新。

**验收**：
- 三条链路各有 1 份脱敏日志；
- 花费账本不超过 N3 上限；
- `make lint test check-numbers` 通过。

### Phase 13 — 合成流水线真实执行（最小规模，受 N3 约束）

1. 先看 RECON 中 `gen scenario` 对 `EMBEDDING_OPENAI_API_KEY` 的依赖：
   - 有可用的 embedding 端点：从 `--scenarios 1` 开始完整执行；
   - 没有：跳过场景生成，以官方的 1 个场景描述为输入，从 `gen task` 开始执行。先确认 CLI 支持这样做；不支持就停下报告，不要自己绕过。
2. **执行**：运行 `workbench synth run --scenarios 1 --execute`，产物放在 `data/synth/<run_id>/`，manifest 中标记 `origin: local-synth`。验证：
   - 断点续跑：故意中断一次后续跑；
   - 缓存命中；
   - 账本记账；
   - 结束后自动执行的 `check_all` 验证报告。
3. **价格表**：`configs/pricing.yaml` 中所用模型的单价，必须来自服务商官方价格页，写明 URL 和日期；查不到就保持占位值并标注。
4. 合成出的环境**不得用于任何训练**，也不得与官方数据混合（R2、ADR-011）。

**验收**：U9 更新为已验证（附日志），或如实写明失败原因。

### Phase 14 — Docker 构建与部署验证

1. 执行 `docker compose build`，然后 `docker compose up`（不带 gpu profile），在容器内完成一次 mock 演示："查询 → 写操作 → 审批 → 完成"。
2. 记录镜像大小、冷启动耗时。这两项属于工程事实，必须由脚本实测，并写明测量命令（R3）。
3. 如果构建失败，只修复 Dockerfile 或 compose，补 ADR；不得删减功能来换取构建通过。

**验收**：U6 更新为已验证（附日志）。

### Phase 15 — GPU 验证（可选，在 AutoDL 等 GPU 机器上执行；没有 GPU 就整阶段跳过）

1. **vLLM 服务**：
   - 按 Phase 10 核实过的模型卡信息，确定 Arctic-AWM-4B 的 vLLM 参数。
   - RECON 推断 `awm agent` 从文本中解析工具调用，不需要 `--tool-call-parser`，这一点要在真实服务上确认。
   - 执行 `scripts/serve_vllm.sh`，确认服务可用。
2. **vLLM 后端链路**：用 `WORKBENCH_LLM__BACKEND=vllm` 执行 1 次 `workbench agent run`（N2）。
3. **训练环境**：
   - 初始化 AgentFly 的嵌套 verl submodule（RECON 记录其 URL 为 SSH 形式，需要时改用 HTTPS 克隆到本地，但不得修改 `.gitmodules` 中上游的定义）；
   - `cd train && uv sync`；
   - 运行 `workbench train preflight`。
4. **smoke 训练**：执行 `workbench train launch --execute`（≤5 step），产物目录标记 `NO_RESULTS`。只记录"rollout → 奖励 → 更新"循环是否跑通，以及显存峰值和耗时这类工程事实，**不记录也不解读任何 reward 曲线的好坏**。
5. U1、U3、U4、U5 按实际结果更新。

**验收**：每个子项都有日志，或者有如实写明的失败原因和阻塞点。

---

## 4. 收尾要求（最后一个执行的 Phase 结束时完成）

1. LIMITATIONS.md 只保留仍未验证的项目；已验证的项目移到新的"已验证"小节，附日期、机器类型（不写主机名）和日志路径。
2. README 的"GPU 部署路径"与"结果说明"两节，按实际验证状态改写。
3. CHANGELOG 记录本轮变更。
4. 按 TASK.md §6 的最终验收清单重新逐条执行，并贴出输出。
5. 按 N5 的授权决定是否开 PR；没有授权就只推送分支。

---

## 5. 停止条件（在 TASK.md §7 基础上新增）

- 本机无法访问 huggingface.co 或 arxiv.org
- 论文原文与 §2.2 转录不一致
- 模型卡许可证与用途冲突
- 预计 API 花费将超过 N3 上限
- 某个验证只能通过修改上游、修改官方数据或放宽安全策略才能完成
- 需要执行 N5 中我没有勾选的操作
