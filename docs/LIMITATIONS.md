# LIMITATIONS — 未验证项与已知限制

本文件如实列出本仓库**没有做到**或**没有验证**的部分（R8）。标注 UNVERIFIED-LOCAL 的项目在开发沙箱（无 GPU、无 Docker daemon、`huggingface.co` 与 `arxiv.org` 被出口策略拦截）中无法验证；表中写明了验证所需的资源和步骤。

## 1. UNVERIFIED-LOCAL 项

| # | 项目 | 本仓库提供了什么 | 为什么没验证 | 验证所需资源与步骤 |
|---|---|---|---|---|
| U1 | 用 vLLM 服务 Arctic-AWM | `configs/serving/arctic-awm-4b.yaml`、`scripts/serve_vllm.sh`、`workbench serve vllm-cmd`、compose 中的 `vllm` 服务（`gpu` profile） | 无 GPU；无法下载权重；未能阅读模型卡，因此 tool-call parser 与 chat template 默认不设置 | Linux x86_64 + CUDA GPU、HF 访问。执行 `scripts/serve_vllm.sh`，再用 `WORKBENCH_LLM__BACKEND=vllm` 运行 `workbench agent run` |
| U2 | 真实 LLM 驱动的智能体 | `openai_compat` / `vllm` 后端（流式、超时、重试），单测覆盖协议层 | 所有端到端测试都使用 mock 回放 | 一个 OpenAI 兼容端点与 key（见 `.env.example`） |
| U3 | train 环境安装 | `train/pyproject.toml`、`train/uv.lock`（只锁定；`uv lock --check` 通过） | 需要 CUDA；`flash-attn` 构建依赖 torch | GPU 机器上执行 `cd train && uv sync` |
| U4 | veRL 嵌套子模块与 Hydra 组合 | `check_override_keys`（静态键检查，曾在 scratch 中针对 fork @001f000 实测）、`hydra_compose_check` | 嵌套子模块默认不初始化（SSH URL）；Hydra 只在 train 环境中存在 | `git -C third_party/AgentFly submodule update --init verl`，然后 `workbench train preflight` |
| U5 | smoke 训练运行 | `configs/train/smoke.yaml`、`workbench train launch --execute`（产物标 `NO_RESULTS`） | 无 GPU | 单卡 CUDA GPU（preflight 默认要求至少 12000 MiB 可用显存）+ U3 + U4 |
| U6 | Docker 镜像构建与 compose 启动 | `Dockerfile`、`docker-compose.yml`（`docker compose config` 已通过） | 沙箱中没有 Docker daemon | 有 Docker 的机器上执行 `docker compose up --build` |
| U7 | 官方数据集下载与接入 | `scripts/download_data.sh`、`make data`、`workbench env search` 等按官方格式读取 | `huggingface.co` 被拦截 | HF 访问；`make data` 后运行 `workbench doctor`，再用官方场景跑 `workbench env up` |
| U8 | `awm agent` / `awm verify` 各跑一次单任务 | 集成测试用真实 AWM 代码跑通了建库、server、MCP 调用和 `check_all`；UI 的轨迹查看器能读取 `awm agent` 的输出格式 | 两个命令都需要 LLM 端点，`verify --mode sql` 还需要 LLM key | 一个 LLM 端点；在官方或迷你场景上各执行一次，并显式传 `--temp_server_path`、`--db_path`、`--output_dir` |
| U9 | 合成流水线真实执行 | `workbench synth run --execute`，含 checkpoint、缓存、账本 | 需要 LLM 与 embedding API key；未在本仓库执行过，账本中的价格是占位值 | API key（见 `.env.example`）；小规模试跑 `--scenarios 1` |

CI（`.github/workflows/ci.yml`）已在 GitHub Actions 上运行并通过（run 9，提交 `bbb541c`），因此不列为未验证项。此前 run 4–8 失败，原因分别是 detect-secrets 误报和 loguru 在 CI 中强制彩色输出，均已修复。

## 2. 数字与许可证

- **论文数字已核对（2026-09-24）**：registry 中 30 条均已对照 arXiv 2602.10090 v3 Table 4 的 HTML 与 PDF 逐格核对，`verified: true`，证据见 `docs/verification/2026-09-24-paper-table4.md`。仍然成立的限制：
  - 这些是论文报告值，本仓库没有复现，也不能复现（R1）；
  - 只登记了 Base 与 AWM 两行的 5 列，Table 4 的其它列与 Simulator、EnvScaler 两行未登记；
  - 发布的 Arctic-AWM-4B / 8B / 14B 权重是否就是 AWM 行所评测的模型，模型卡没有明说，registry 中保持"推定"；8B、14B 模型卡的 `base_model` 元数据写成了 Qwen/Qwen3-4B，与卡片正文矛盾（`docs/verification/2026-09-24-model-cards.md`）。
- **AWM 没有许可证**：2026-09-24 复查，上游 HEAD 仍为 `85e322f`，仍无 LICENSE；第三方 PR #17 仍未合并（`docs/verification/2026-09-24-licenses.md`）。本仓库不包含 AWM 代码副本，也没有 patch；但运行时 import 并执行 AWM 代码，这一点的法律判断由仓库主人负责（ADR-003）。询问许可证的 issue 草稿见 `docs/verification/awm-license-issue-draft.md`，尚未发送。
- **数据集与模型许可证已核实**：数据集 CC-BY-4.0（需署名，见 README 与 UPSTREAM §5），三个模型 Apache-2.0。

## 3. 训练配方的公开状态

Phase 0 结论为 **(b)**：上游只公开了环境适配（OpenEnv 的 `agent_world_model_env`），没有公开完整的训练配方（超参数、奖励设计、数据配比）。因此：

- 不创建 `paper_mirror` profile（ADR-012）；
- smoke 配置只演示"rollout → 奖励 → 更新"链路，使用 AgentFly 自带的 `calculator` 工具与数学奖励，**不接触 AWM 环境**；
- 本仓库没有、也不能复现 Arctic-AWM 的训练。

## 4. 应用层未经评测

- 本仓库从未对应用层（网关、审批、守卫、记忆、prompt）做过任何效果评测，也没有跑过 `awm bench` 或任何评测 harness（R1，ADR-013）。
- 测试证明的是"机制按设计工作"（例如写操作一定要求审批、令牌只能用一次、守卫会终止循环），不代表任务完成情况会因此变好或变坏。
- mock 回放脚本（`tests/fixtures/trajectories/*.jsonl`）全部为手写，不是模型输出。

## 5. 合成环境未用于训练

`data/synth/` 下的任何产物（manifest 标 `origin: local-synth`）都没有被用于任何训练，也没有与官方数据混合（ADR-011）。实际上本仓库从未真实执行过合成（U9）。

## 6. 其它已知限制

- **迷你夹具的工具名是暂定的**：`tests/fixtures/awm_mini` 的工具名取自 OpenEnv 对 `e_commerce_33` 的抓取记录（RECON §1.6），无法与官方数据集比对。夹具只借用 AWM 的数据格式。
- **风险分级是启发式的**：按工具名动词分级，未知动词默认按 `write` 处理（需要审批）。`workbench gateway export-risk` 导出的表需要人工复核。
- **每次工具调用新建一个 MCP session**：与 AWM 的做法一致，未做连接池（`docs/IDEAS.md`）。
- **单进程部署**：审批令牌的"已使用"集合、限流桶、忙碌集合都在进程内存中；多副本部署需要共享存储。
- **长期记忆的 TTL 精度为秒级**（LangGraph `SqliteStore` 的实现）。
- **上游缺陷未修复**：veRL fork @001f000 中残留合并冲突标记（RECON §10）；AgentFly 示例脚本使用了 fork 中不存在的 Hydra 键。按 R5 不修改上游，仅在文档中记录，smoke 配置绕开了这些问题。
- **沙箱的 PID 1 不回收僵尸进程**：compose 中设置了 `init: true`；直接在类似环境中运行时，进程组清理后可能残留僵尸条目（不占资源）。
