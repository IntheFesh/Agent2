# CLAUDE.md — BizAgent Workbench 工作守则（R1–R14 要点；完整版见 TASK.md）

边界：只改"怎么用、怎么部署、怎么管、怎么看"，不改"模型有多强"；本仓库不产生任何模型性能数字。

- **R1 禁止评测**：不跑 `awm bench`（任何 mode），不跑 BFCLv3 / τ²-bench / MCP-Universe 的 harness，不批量跑任务后汇总任何比率。只允许 `awm agent`、`awm verify` 各跑一次单任务，用来证明链路打通，结果不写成比率或效果结论。
- **R2 不改被评测对象**：
  - 不改 AgentWorldModel-1K 官方数据与上游训练配置；自合成的环境只放 `data/synth/`，manifest 标 `origin: local-synth`。
  - 不跑完整 RL，只允许 smoke 配置（≤1.7B、LoRA、≤5 step，产物目录标 `NO_RESULTS`）；不用自训权重冒充 Arctic-AWM。
- **R3 数字纪律**：
  - 性能数字只能来自 `results/registry.yaml`，每条带 `source` 与 `verified`。
  - 引用处必须注明："论文报告值，由官方模型在官方评测 harness 上测得，不是本仓库应用层的测量结果。"
  - 应用层未经评测：禁用"准确率提升 / 成功率 / 可靠性提升 X%"之类表述，只写机制。
  - 自测数字只限工程事实，并写明测量命令。
- **R4 反幻觉**：用任何上游函数、CLI 参数、配置键之前先读源码，在 `docs/RECON.md` 记"文件:行号"。vLLM parser 与 chat template、LangGraph、MCP SDK 以已安装版本的源码为准，并记录版本号。找不到就停下报告，不猜，不自造同名接口。
- **R5 上游隔离**：
  - 上游只作为 `third_party/` 下固定 SHA 的 submodule，禁止改其中文件；确需改动时用 `patches/NNNN-<desc>.patch`，并在 `docs/DECISIONS.md` 加 ADR。
  - 自有代码全部放在 `src/workbench/`；`docs/UPSTREAM.md` 精确到文件级。
- **R6 许可证**：许可证记入 UPSTREAM.md；缺失或冲突即停止并报告。数据集与权重不入库，只提供下载脚本。
- **R7 密钥**：不提交任何 key；提供 `.env.example`；pre-commit 中加入 gitleaks 或 detect-secrets。
- **R8 诚实**：
  - 测试真实执行，报告里贴命令与输出结尾；失败如实报告。
  - 不用空实现或固定返回值冒充功能可用；未完成项写入 `docs/LIMITATIONS.md`。
  - 本地无法验证的 GPU / 外部 API 步骤标 `UNVERIFIED-LOCAL`，并写明验证所需资源。
- **R9 提交**：
  - 每个子任务至少一个小提交，Conventional Commits（feat / fix / docs / test / chore / refactor），英文写明做了什么、为什么。
  - 每个提交在 mock 模式下都能通过 `make test`；不 force push，不改写历史，不改 git 作者配置。
- **R10 无 GPU**：自有代码在 CPU + mock LLM 下跑通测试；GPU 相关部分只写脚本和文档。
- **R11 依赖隔离**：app（根目录）与 train（`train/`）是两个独立的 uv 环境；不为共存而升级或降级上游固定版本；app 代码不得 import torch 分布式、veRL、AgentFly 训练模块。
- **R12 阶段闸门**：每个 Phase 结束时按 TASK.md §5 输出报告，然后停下，等"继续"。
- **R13 范围**：不加任务书以外的大功能，新想法写进 `docs/IDEAS.md`。
- **R14 语言**：文档用中文（README 开头附 5 行英文摘要）；代码标识符和注释用英文；提交信息用英文。

冲突处理：任务书与上游不一致时以上游源码为准，停下报告。停止条件见 TASK.md §7。

当前状态（Phase 0–8 全部完成，TASK_v2 的 Phase 9–12 已完成；入口见 `README.md`，未验证项见 `docs/LIMITATIONS.md`）：
- 上游固定在 AWM `85e322f`、AgentFly `1256586`。嵌套的 `verl` 默认不初始化；`mcp-adapted-bench` 永不使用；`patches/` 为空。
- 训练配方结论为 (b)：只有环境适配，没有完整官方配方，因此不创建 `paper_mirror` profile；smoke 只用 AgentFly 自带工具与奖励（ADR-012）。
- AWM 仓库没有许可证：按 ADR-003 只引用、不复制、不打补丁（2026-09-24 复查仍无）。数据集 CC-BY-4.0（须署名），模型 Apache-2.0。
- registry 30 条已对照 arXiv v3 Table 4 核对（`verified: true`，证据在 `docs/verification/`）；模型身份仍为推定。Phase 9–15 按 `TASK_v2.md` 在 `phase9-verification` 分支进行。
- 官方数据集用 `make data` 下载到 `data/awm1k`（revision `dde80a0`，不入库）；依赖它的测试标 `official_data`，无数据时自动 skip。迷你夹具的接口已与官方 `e_commerce_33` 对账。
- 调用 AWM 时必须显式传 `--temp_server_path`、`--db_path`、`--output_dir`；合成前先复制种子文件。否则会写入官方数据目录或 submodule。
- AWM server 以进程组启动，必须用 `killpg` 回收；所有命令设置 `PYTHONPYCACHEPREFIX`，避免在 submodule 中留下 `__pycache__`。
- 真实 LLM 用 DeepSeek `deepseek-flash`（base URL `https://api.deepseek.com`）；key 只从 `DEEPSEEK_API_KEY` 读取，只在进程环境里映射给 `OPENAI_API_KEY` 等变量。每次外部调用前先估算并记入 `docs/verification/cost-ledger.md`（上界口径）。Phase 12 的三条链路各已执行 1 次，不得重跑（N2）。
- `awm agent` 只用 `--mcp_url` 模式连接 env-manager 会话：`--scenario` 自动起服结束时必抛 `SameFileError`，还会把服务代码写到 `--envs_path` 目录。
- 改动 README 或 docs 后运行 `make check-numbers`；改动 registry 后运行 `make results`。
