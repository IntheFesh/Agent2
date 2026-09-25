# LIMITATIONS — 未验证项与已知限制

本文件如实列出本仓库**没有做到**或**没有验证**的部分（R8）。

- §1 是验证状态，分两节：1.1 为仍未验证的项目（UNVERIFIED-LOCAL），写明原因与验证所需的资源和步骤；1.2 为已在真实环境中验证过的项目，附日期、机器类型与日志路径。
- §2–§6 是已知限制，与是否验证无关。

开发环境最初拦截了 `huggingface.co` 与 `arxiv.org`，2026-09-24 起放开（TASK_v2 Phase 9）。本文件的状态截至 2026-09-25：TASK_v2（Phase 9–15）收尾，以及 polish-v3 的 Phase 16–17（审批前预演的限制见 §6）。

## 1. 验证状态

### 1.1 未验证

U1、U3、U4、U5、U8 都需要 GPU 机器，原因相同：**Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行**（`user-decisions.md` D22）。以后仓库主人在新分支上单独执行 Phase 15，回贴日志后另提一个小 PR。U9 的原因不同，见表。

| # | 项目 | 本仓库提供了什么 | 为什么没验证 | 验证所需资源与步骤 |
|---|---|---|---|---|
| U1 | 用 vLLM 服务 Arctic-AWM | `configs/serving/arctic-awm-4b.yaml`、`scripts/serve_vllm.sh`、`workbench serve vllm-cmd`、`workbench serve probe`、compose 中的 `vllm` 服务（`gpu` profile）。按 D11（ADR-020），profile 启用 `--enable-auto-tool-choice --tool-call-parser hermes`，compose 已按 D16 同步，由单测防止漂移。依据是读源码：模型 chat template 的 `<tool_call>{...}</tool_call>` 格式与 vLLM v0.19.0 `hermes` parser 的解析格式一致；不带 `tools` 的请求（`awm agent` 的文本协议）`tool_choice` 保持 `"none"`，parser 不参与（RECON "Phase 12.5 报告之后"） | Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行。待在真实服务上确认：act 请求能否得到原生工具调用；没有设置 reasoning parser，思考内容中出现 `<tool_call>` 时会不会被误解析 | Linux x86_64、24 GB CUDA 显卡（驱动支持 CUDA ≥ 12.8）、Hugging Face 或其镜像站。runbook §3.3–§3.6：启动服务，`workbench serve probe` 各发 1 个原生 tools 请求和 1 个文本协议请求，再用 `WORKBENCH_LLM__BACKEND=vllm` 运行 1 次 `workbench agent run` |
| U3 | train 环境安装 | `train/pyproject.toml`、`train/uv.lock`（只锁定；`uv lock --check` 通过）；flash-attn 的构建环境固定使用锁定的 torch（ADR-027） | Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行。安装需要 CUDA；flash-attn 只有源码包，也没有对应 torch 2.10.0 的预编译 wheel，要在本机用 nvcc 编译 | runbook §3.1（15A，不装 flash-attn）与 §4.2（15B，完整安装：A100 级显卡、nvcc 12.x、至少 64 GB 内存） |
| U4 | veRL 嵌套子模块与 Hydra 组合 | `check_override_keys`（静态键检查，曾在 scratch 中针对 fork @001f000 实测）、`hydra_compose_check`。经 HTTPS 初始化嵌套子模块的命令已在开发容器中对 AgentFly 的本地克隆原样执行过（RECON "Phase 15 runbook"），但 Hydra 组合没有在 train 环境中执行过 | Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行。Hydra 只存在于 train 环境中（依赖 U3） | runbook §4.1（经 HTTPS 克隆，不改 `.gitmodules`）与 §4.4（`workbench train preflight`） |
| U5 | smoke 训练运行 | `configs/train/smoke.yaml`、`workbench train launch --execute`（产物标 `NO_RESULTS`），训练进程只拿白名单环境变量（ADR-026） | Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行 | 单卡 CUDA GPU（preflight 要求至少 12000 MiB 可用显存）+ U3 + U4。runbook §4.6：只记录是否跑通、峰值显存与耗时 |
| U8 | `awm agent` / `awm verify` 各跑一次单任务 | 2026-09-24 用 DeepSeek（`deepseek-flash`）各执行过 1 次：`awm verify --mode sql` 链路打通；`awm agent` 的 LLM 调用、文本解析与 MCP 工具清单打通（见 1.2）。`workbench verify`（ADR-025）让 `awm verify` 拿不到 key | **部分验证**：`awm agent` 第 2 轮 DeepSeek 在纯文本中输出了它自己的 DSML 工具调用标记，AWM 只识别 `<tool_call>`（`awm/core/agent.py:130-167`），循环提前结束，没有执行写操作；按 D3 不改上游、不写适配器。剩下的一半需要一个原生遵循 `<tool_call>` 文本协议的端点：Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行 | 经 vLLM 服务的 Arctic-AWM（U1），用 `--mcp_url` 模式再执行一次（`--scenario` 自动起服有上游缺陷，见 §6）。runbook §3.6–§3.7：`awm agent` 连本机 vLLM，`awm verify` 用 `--mode code`（不需要裁判） |
| U9 | 合成流水线的 `gen scenario` 步骤 | `workbench synth run` 的完整模式（不加 `--scenario-file`，从 `gen scenario` 开始） | 本轮没有 embedding 端点：DeepSeek 不提供 embedding 接口（ADR-021）。其余 6 个步骤已真实执行（见 1.2） | 一个 OpenAI 兼容的 embedding 端点（AWM 默认用 `text-embedding-3-large`，`awm/core/scenario.py:40-43`）与 `EMBEDDING_OPENAI_API_KEY`；用 `--scenarios 1` 试跑完整模式 |

### 1.2 已验证

以下项目已在真实环境中执行过。其中的数字都是单次测量的工程事实；链路演示只证明"打通"，不构成评测（N2）。机器类型只写平台，不写主机名。

- **CI**：`.github/workflows/ci.yml` 在 GitHub Actions 上执行 lint、secrets、test、number guard 与 doctor 并通过。Phase 8 末的 run 9（提交 `bbb541c`）首次通过，此前 run 4–8 的失败（detect-secrets 误报、loguru 在 CI 中强制彩色输出）都已修复；Phase 9 核对 `f140123` 的 run 10 通过。本轮每次推送后都检查了 CI，失败过的 run 27（测试依赖时间窗口）与 run 29（detect-secrets 拦下日志中的提交哈希）都已修复。
  - 日期：2026-09-24 起；机器：GitHub 托管 runner（`ubuntu-latest`）；证据：`docs/verification/2026-09-24-preflight.md` §2。本轮最后一次提交的运行结果见 `phase9-verification` → `main` 的 PR。
- **Docker 镜像构建与 compose 启动（原 U6）**：
  - 本机 `dockerd` 能启动，但 Docker Hub 返回 429，按 D15 改在 GitHub Actions 托管 runner 上验证。
  - `.github/workflows/docker-smoke.yml` 运行 `scripts/docker_smoke.py`（当时在 push 到 `phase9-verification` 与 `main` 时触发；2026-09-25 起改为 push 到 `main` 与以 `main` 为目标的 PR，D27）：构建，启动（不带 `gpu` profile），经 HTTP API 走完"查询 → 写操作 → 审批 → 完成"，最后停止。2026-09-24 的两次运行都通过（run 1 提交 `5419a2c`，run 2 提交 `c2a8a08`）。
  - 只有一个约 880 MB 的镜像，`app` 与 `env-manager` 共用；构建分别用了 15.9 s 与 23.0 s，冷启动 13.4 s 与 15.0 s。这些是单次测量，随 runner 变化。
  - `gpu` profile 仍未验证（U1）。
  - 日期：2026-09-24；机器：GitHub 托管 runner（`ubuntu-24.04`）；证据：`docs/verification/2026-09-24-docker-smoke.md`，日志 `docs/verification/logs/2026-09-24-phase14-docker-smoke.log`。
- **官方数据集下载与接入（原 U7）**：`make data` 匿名下载 revision `dde80a0`，8 个文件齐全，条目数与数据集卡一致；`workbench doctor` 的 dataset 一项为 ok；官方 `e_commerce_33` 经 env-manager 真实启动，`list_tools` 返回 39 个工具。
  - 日期：2026-09-24；机器：Claude Code 云端容器（Linux x86_64，无 GPU）；证据：`docs/verification/2026-09-24-dataset.md`，日志 `docs/verification/logs/2026-09-24-phase11.log`。
- **真实 LLM 驱动的智能体（原 U2）**：`openai_compat` 后端接 DeepSeek（`deepseek-flash`，key 只从环境变量读取），`workbench doctor` 的 llm 一项为 ok；在官方 `e_commerce_33` 任务 0 上执行 1 次 `workbench agent run`，走完"规划 → 读工具 → 审批 → 写工具 → 回答"，DB diff 为 `cart_items` 新增 1 行。Phase 12.5 的修复完成后，在同一任务上用默认预算再执行了 1 次（提交 `f135189`，WALKTHROUGH §5.3），链路仍然打通。vLLM 后端仍未验证（U1）。
  - 日期：2026-09-24；机器：Claude Code 云端容器（Linux x86_64，无 GPU）；证据：`docs/verification/2026-09-24-llm-chain.md`，日志 `docs/verification/logs/2026-09-24-phase12-workbench-agent.log` 与 `docs/verification/logs/2026-09-24-phase12.5-workbench-agent.log`（各附 trace）。
- **`awm agent` 与 `awm verify --mode sql` 的单次执行（U8 已验证的一半）**：`awm verify` 执行官方 verifier 并由 DeepSeek 裁判，写出 `verify.sql.json`；`awm agent` 的 LLM 调用、文本解析与 MCP 工具清单打通；UI 轨迹查看器能渲染这次 `awm agent` 的真实 `trajectory.json`。
  - 日期：2026-09-24；机器：Claude Code 云端容器（Linux x86_64，无 GPU）；证据：`docs/verification/2026-09-24-llm-chain.md` §4–§6，日志 `docs/verification/logs/2026-09-24-phase12-awm-agent.log`、`2026-09-24-phase12-awm-verify.log`、`2026-09-24-phase12-ui-viewer.log`。
- **合成流水线真实执行（原 U9 除 `gen scenario` 外的部分）**：用 DeepSeek（`deepseek-flash`）经本地代理执行 1 次 `workbench synth run --scenario-file … --execute`，场景为手写的 `local_it_service_desk`（D5）。结果：
  - 从 `gen task` 到 `gen verifier` 的 6 个步骤都一次成功；
  - 故意中断后用同一命令续跑，6 个步骤按 checkpoint 跳过，只重做被中断的验证；
  - 缓存命中另用一次不带 key 的重放验证；
  - 账本逐条记账，上界口径 ¥1.2240；
  - `check_all` 报告 1 个环境启动成功、15 个工具。
  - 日期：2026-09-24；机器：Claude Code 云端容器（Linux x86_64，无 GPU）；证据：`docs/verification/2026-09-24-synth.md`，日志 `docs/verification/logs/2026-09-24-phase13-synth.log`。
- **合成 runner 在步骤中途被中断后续跑（Phase 14，零成本）**：用真实 AWM 的 gen 步骤与本地回放的上游重做 Phase 13 的中断。中断落在 `gen env` 中间，没有进程残留；同一命令续跑后 `env` 由缓存重放，确定性的输出与 Phase 13 一致（`db_path` 除外）。预算熔断与上游错误的判定由离线测试覆盖（ADR-023、ADR-024）。
  - 日期：2026-09-24；机器：Claude Code 云端容器（Linux x86_64，无 GPU）；证据：`docs/verification/2026-09-24-phase14-synth-resilience.md`，日志 `docs/verification/logs/2026-09-24-phase14-synth-resilience.log`。

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

`data/synth/` 下的任何产物（manifest 标 `origin: local-synth`）都没有被用于任何训练，也没有与官方数据混合（ADR-011）。唯一一次真实合成（Phase 13，`data/synth/p13_it_service_desk/`，1 个手写场景）同样如此：它的场景名以 `local_` 开头、不与任何官方场景重名（ADR-021），产物不入库。

## 6. 其它已知限制

- **迷你夹具只是官方接口的最小子集**：2026-09-24 已与官方 `e_commerce_33` 对账，夹具 7 个工具的名称、参数名、必填字段与顶层返回字段与官方一致（`docs/verification/2026-09-24-fixture-reconciliation.md`），不再是"暂定"。但夹具只有 7 个工具（官方 39 个），参数、表和列都大幅精简，样例数据、任务与 verifier 为手写；它用来测试机制，不代表官方环境的行为全貌。
- **上游生成代码的语义缺陷网关无法识别**：在官方 `e_commerce_33` 上观察到，查询不存在的商品 ID 会返回 `id: 0` 的占位对象而不是错误，向购物车加入不存在的 offer ID 也会成功写入。网关只能把它们判为 `ok`（ADR-014），这类问题只能靠审批与 DB diff 暴露。
- **"空结果"判定是启发式的**：只对 `read` 级工具生效；包装对象中所有列表字段都为空、且没有嵌套对象时判为 `empty`（ADR-014）；形态不同的返回可能被判为 `ok`。写类工具成功时一律为 `ok`。POST/PUT/PATCH/DELETE 工具因 HTTP 方法下限不会再被判为 `read`（ADR-015）；用 GET 实现的写操作仍可能被标成 `empty`，需要在 `tool_policy.yaml` 中 override。已在官方 `e_commerce_33`、`social_media_4` 与迷你夹具上验证。
- **风险分级是启发式的**：按工具名动词分级，未知动词默认按 `write` 处理（需要审批）；自 Phase 12.5 起再以路由的 HTTP 方法为下限（DELETE 至少 `destructive`，POST/PUT/PATCH 至少 `write`，方法取自离线目录，ADR-015）。官方数据集上 POST/PUT/PATCH/DELETE 工具中被判为 `read` 的从 38 个降到 0 个（`docs/verification/logs/2026-09-24-phase12.5-risk-floor.log`）。仍然存在的限制：用 GET 实现的写操作无法识别；以 POST 实现的纯查询现在也需要审批（可用 `overrides` 调低）；`overrides` 按原样生效，即使低于下限；目录里查不到的工具（场景不在数据目录中、代码无法解析、路径或 operation_id 不是字面量）仍只按名称判断。`workbench gateway export-risk` 导出的表（含 `http_method` 与 `heuristic_risk` 列）仍需人工复核。
- **DeepSeek 思考模式的 `reasoning_content` 回传**：Phase 12.5 按官方文档实现（ADR-017）：带 `tools` 的请求会补上此前各轮的 `reasoning_content`。文档没写清、本仓库没有猜测的情况：没有 `reasoning_content` 的历史轮次（非思考模式产生的、来自其它后端的、进程重启后丢失的）原样发送，服务端怎么处理未知；API 参考只把请求中的 `reasoning_content` 写成 Chat Prefix Completion（Beta）的输入，与思考模式指南不一致。其它限制：记录只在进程内存中，服务重启后从 checkpoint 恢复的会话可能按文档收到 400；每个后端实例最多记 1024 条；非 system 历史完全相同的两个会话共用一条记录。2026-09-24 实测 DeepSeek 对缺少 `reasoning_content` 的请求仍返回 200，因此"不回传就 400"这一规则只在单测的假服务端上验证过。
- **官方多工具场景的 token 开销**：Phase 12 发现 act 节点把工具定义放了两遍，每次 act 约 26K token。Phase 12.5 改为只经原生 `tools` 参数传定义（ADR-018），离线测得同一次 act 调用从 25640 降到 14358 token；默认 `agent.token_budget` 按实测改为 240000（39 个工具的官方场景上跑满 12 次 act 仍在预算内）。仍然存在的限制：plan 请求仍带完整定义（约 12K token）；预算默认值只按一个参考场景和一次运行测得。
- **`llm.max_tokens` 含思考 token**：DeepSeek 把思考 token 计入输出 token（探针 P3）。Phase 12 实测单次调用的输出最多约 960 token，默认值按其 8 倍改为 8192（ADR-018）；更难的任务可能需要更长的思考。
- **AWM 的输出上限参数对 DeepSeek 不生效**：`awm agent` 与 `awm verify` 发送 `max_completion_tokens`（`awm/core/agent.py:370`、`awm/core/verify.py:309`），DeepSeek 接受但忽略（探针 P4），因此这两个命令对 DeepSeek 没有客户端输出上限。
- **审计脱敏是启发式的**：Phase 12 发现 `PHONE` 正则会把带微秒的时间戳（`17:05:31.506430`）和空格分隔的时间戳（`2026-09-23 17:05:16`）的一部分替换成 `[PHONE]`，Phase 12.5 已收紧正则修复（ADR-016）。仍然存在的限制：紧跟在"数字:"之后、或后面紧接":数字"的电话号码不会被脱敏；非 ISO 格式的日期（例如 `24.09.2026`）仍可能被当成电话号码；脱敏只作用于审计摘要，trace 中的工具结果不做脱敏。
- **执行生成代码的子进程与密钥**：仓库主人在 Phase 12.5 指出，env-manager 用完整的进程环境启动 env server，生成代码能读到 `DEEPSEEK_API_KEY`。Phase 12.5 起改为白名单（ADR-019）：
  - env server、`awm env reset_db`、`awm env check_all` 只拿到 `PATH`、locale、`TMPDIR` 等运行必需的变量，看不到任何 key；
  - gen 步骤经本地代理时只拿到占位 key。
  - 测试会启动真实子进程，并读取真实 AWM 进程组的 `/proc/<pid>/environ`，断言其中没有 `DEEPSEEK_API_KEY` 和任何 `*_API_KEY`（`docs/verification/logs/2026-09-24-phase12.5-subprocess-env.log`）。

  仍然存在的限制：
  - **`awm verify` 要经 `workbench verify` 运行**（Phase 15 起，ADR-025）：
    - `awm verify` 在同一进程里执行 verifier 代码（namespace 中直接提供了 `os`，`awm/core/verify.py:104-126`、`:151-174`），sql 模式下还从同一进程的环境读取裁判的 key（`:416-433`）；
    - `workbench verify` 让 code 模式的进程拿不到任何 key，sql 模式的进程只拿到代理地址与占位 key；
    - 直接运行 `awm verify` 时，shell 里的 key 仍会交给 verifier 代码；
    - sql 模式下，verifier 代码仍能经代理发起 LLM 调用。调用记入 `verify_ledger.jsonl`，但没有预算上限。
  - **`gen env`、`gen verifier` 的生成代码**与调用 LLM 的步骤在同一进程树里。
    - 经代理时，生成代码只能看到占位 key 和代理地址。它仍能经代理发起 LLM 调用，费用记入账本。
    - 不经代理时（只有在 Python 中直接调用才会出现），会看到真实的 `OPENAI_API_KEY`。
  - **只隔离了环境变量**。子进程与 workbench 以同一用户运行，仍能读取该用户可读的文件，例如 `.env`、`~/.aws/credentials`。要隔离这些文件，需要另一个用户或容器沙箱。
  - **白名单是固定的**，没有配置开关。
  - **`awm agent --scenario` 自动起服**时，server 会继承 `awm agent` 的环境，其中含 key（`awm/core/server.py:174-195`）。本仓库只用 `--mcp_url` 模式。
  - **训练进程**（Phase 15 起，ADR-026）：`workbench train launch` 与 preflight 的探针只拿到 `train_env()`，即基础变量、代理设置，以及 CUDA、NCCL、PyTorch、vLLM、Ray、veRL、Hugging Face 等按名称或前缀放行的变量，不含任何名字像凭据的变量。仍然存在的限制：
    - 按前缀放行比只按名称放行宽；
    - 白名单是否够用只能在 GPU 机器上验证（U5）。缺了变量时，可以加到 `train.env_passthrough`；
    - smoke 训练的 `calculator` 工具用 sympy 的 `sympify` 解析模型输出（`agentfly/tools/src/calculate/tools.py:20`），`sympify` 内部使用 `eval`（sympy 1.14.0 `sympy/core/sympify.py:138-139`）。训练进程读不到凭据类环境变量，但仍能读到该用户可读的文件。
  - **Phase 12 的 env server（官方 `e_commerce_33`）是在修复前启动的**，当时进程环境里有 `DEEPSEEK_API_KEY`。离线用正则扫描了官方全部 1000 个场景的代码（上面日志的 §3），结果如下：
    - 所有场景只按字面名读取 `PORT`、`HOST`、`DATABASE_PATH`，没有整体访问 `os.environ`；
    - `e_commerce_33` 没有 import 任何网络库。

    正则扫描排除不了动态访问。仓库主人决定暂不轮换，自己检查 DeepSeek 后台的用量记录，Phase 13 结束后删除这把 key（`user-decisions.md` D12）。
  - **后续安排**（`user-decisions.md` D13、D18）：两项遗留缺口已作为 Phase 15 的前置修复完成：`awm verify` 经 `workbench verify` 运行（ADR-025），训练改用白名单（ADR-026）。
  - **Phase 15 的 GPU 步骤**（D18、D21、D22）：runbook 已写好（`docs/runbooks/phase15-gpu.md`）。按 D22，本轮不执行，U1、U3、U4、U5、U8 保持未验证（§1.1）；以后仓库主人在新分支上单独执行，回贴日志后另提 PR。
- **合成步骤没有输出上限；预算熔断会在途超支**：
  - AWM 把 `max_tokens` 改名为 `max_completion_tokens` 发出（`awm/gpt.py:160-164`），DeepSeek 忽略这个参数（探针 P4），所以单个请求的输出只受服务端默认上限约束（思考模式 64K token）。
  - Phase 14 起，本地代理按账本累计费用做预算熔断（`synth.budget`，默认 ¥5，ADR-023）。检查发生在转发之前，已经转发、尚未返回的请求照常完成并计费，所以实际花费可能超过上限，超出部分最多是超限那一刻在途请求的费用之和。
  - 代理没有并发上限（IDEAS 5）。
  - 价格表中没有的模型会被拒绝（fail closed），换模型前要先补价格或设 `budget: null`。
  - 熔断只在经代理执行时生效。
- **中断合成运行后的残留**：
  - Phase 14 起，runner 把每个步骤放在独立的进程组里。收到 SIGINT/SIGTERM 时，回收步骤及其全部后代所在的进程组，其中包括 AWM 在独立会话中启动的测试 server（ADR-022）。2026-09-24 用真实 AWM 验证：中断落在 `gen env` 中间，没有进程残留。
  - 仍然存在的限制：
    - AWM 被 SIGTERM 结束时不执行 `finally`，临时目录 `/tmp/env_test_*` 会留下，需要手动删除；
    - runner 自己被 SIGKILL 时无法回收；
    - 找后代依赖 `/proc`，只适用于 Linux；
    - 从收到信号到停止步骤有一段反应时间（实测最长约 0.3 s），其间步骤可能多发出请求，续跑时由缓存重放或重新生成。
  - 未完成的步骤重做前，输出先移到 `attempts/<步骤>.<n>/`。这样 AWM 在 `gen env`、`gen verifier` 中自带的续跑不再起作用；请求正文与上次不同时会再次计费。
- **合成步骤的判定依赖代理的账本**（Phase 15 起，ADR-024，仓库主人决定 D19）：
  - 此前 runner 只看退出码与输出文件。上游持续出错时，AWM 把失败的请求变成空回复（`awm/gpt.py:195-206`），仍以 0 退出，步骤被记为完成：2026-09-24 回放重做中，`gen verifier` 的 50 个请求全部得到 404，10 行结果都没有代码（`docs/verification/2026-09-24-phase14-synth-resilience.md` §3）。
  - 现在代理把所有重试后仍以上游错误结束的请求记入账本（`failed: true`，含状态码或 `network`）。runner 按请求归并：
    - 丢失的请求多于 `synth.max_failed_requests`（默认 0）时，步骤失败，可以续跑；
    - 丢失的请求在 1 到阈值之间时，步骤记为 `done_with_failures`，失败数写在 CLI 输出与 `validation.json` 中。
  - 仍然存在的限制：
    - 只有经代理执行时才能这样判定；
    - AWM 内部的其它失败（解析模型输出失败、生成的代码校验不通过）不是上游错误，仍然只能看退出码与输出；
    - 5xx、网络错误与超时的请求可能已经在上游计费，账本按 0 计；
    - 同一步骤里正文完全相同的两个请求会被当作一个；
    - 阈值大于 0 时，`done_with_failures` 的步骤缺少部分结果，下游照常运行；
    - 收尾的 `check_all` 只检查环境能否启动，不检查 verifier。
- **合成 manifest 每次执行都会重写**：续跑后 `created_at` 是最后一次执行的时间，不是首次创建的时间。
- **每次工具调用新建一个 MCP session**：与 AWM 的做法一致，未做连接池（`docs/IDEAS.md`）。
- **单进程部署**：审批令牌的"已使用"集合、限流桶、忙碌集合都在进程内存中；多副本部署需要共享存储。
- **长期记忆的 TTL 精度为秒级**（LangGraph `SqliteStore` 的实现）。
- **上游缺陷未修复**：veRL fork @001f000 中残留合并冲突标记（RECON §10）；AgentFly 示例脚本使用了 fork 中不存在的 Hydra 键。`awm agent --scenario` 自动起服在结束时必然抛出 `shutil.SameFileError`（工作库本身就是 `<output_dir>/final.db`，结束时又复制到同一路径：`awm/core/server.py:70-82`、`awm/core/agent.py:596-598`），而且会把服务代码写到 `--envs_path` 所在目录、只终止启动器进程；本仓库的演示改用 `--mcp_url` 模式（`docs/verification/2026-09-24-llm-chain.md` §4）。按 R5 不修改上游，仅在文档中记录，smoke 配置绕开了这些问题。
- **沙箱的 PID 1 不回收僵尸进程**：compose 中设置了 `init: true`；直接在类似环境中运行时，进程组清理后可能残留僵尸条目（不占资源）。
- **审批前预演（Phase 17，ADR-029）是尽力而为的结构比对**：
  - 比对只看改动的表、主键和改动的列名。时间列（类型、默认值、列名三条规则）只记录、不比对；主键不是 INTEGER rowid 别名的表，新增行只比对行数。规则依据官方数据集的统计（RECON §10 "Phase 17"），不保证覆盖所有写法：
    - 用随机值给 INTEGER 主键赋值的 server，会被误报为 `preview_mismatch`；
    - 列名不像时间、却由 server 按当前时间写入的列，会被当成普通列比对，可能误报；
    - 误报都会记入审计并在 UI 标出，但不会阻止已经批准的调用。
  - 预演会把生成的代码再执行一次：数据库以外的副作用（外部请求、写文件）会发生两次。官方 1000 个环境中只有 1 个用到 `httpx`，没有 `requests`、`subprocess`、`os.system`（RECON）。
  - 预演之后、真实执行之前，如果会话数据库被别的调用改动，真实改动可能与预演不同（例如新行的主键），比对会如实报 `preview_mismatch`。正常流程中同一会话的调用是串行的，不会出现这种情况。
  - 耗时与资源：每个需要审批的调用都要多等一次影子 server 启动，本机实测总耗时中位数为 3.7 秒（迷你夹具）和 4.3 秒（官方 `e_commerce_33`），日志见 `docs/verification/logs/2026-09-25-preview-timing.log`；超时为 30 秒。同时运行的预演数受 `env.max_previews` 限制（默认 2），多出的排队等待，等待时间计入超时。
  - 预演记录用审批密钥签名：
    - 网关在内存中只保留最近 256 条记录，checkpoint 中的副本在批准时重新验证；
    - 进程重启且没有固定 `WORKBENCH_APPROVAL_SECRET` 时，挂起中、要求预演的审批只能拒绝后重新发起。
  - 独立运行的 `workbench gateway serve` 只有在配置了 `env.manager_url` 时才能预演；否则按默认的 `require_preview`，destructive 调用在那里无法批准。
  - 审批卡片每类最多显示 `approval.preview_max_rows` 行（默认 20），比对使用全部的键。

