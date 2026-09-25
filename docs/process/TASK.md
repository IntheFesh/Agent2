# Claude Code 任务书：BizAgent Workbench（AWM × AgentFly 工程化整合）

> 使用方式：在一个空目录里启动 Claude Code，把本文件保存为 `TASK.md`，然后发送："完整阅读 TASK.md，严格按其执行，从 Phase 0 开始。"建议 Phase 0 使用 Plan Mode。
> 本任务书中的每一条"必须/不得"都是硬性要求，不是建议。

---

## 0. 角色与总目标

你是本仓库的工程负责人。任务是把两个开源项目整合为一个可落地的"企业级 MCP 智能体工作台"：

- **Snowflake-Labs/agent-world-model（下称 AWM）**：1,000 个由 SQLite 数据库驱动、通过统一 MCP 接口暴露工具的合成业务环境；包含合成流水线（`awm gen ...`）、环境管理（`awm env ...`）、单任务 agent 演示（`awm agent`）、校验（`awm verify`），以及已训练模型 Arctic-AWM-4B/8B/14B 和论文评测结果。
- **Agent-One-Lab/AgentFly**：基于 veRL 的多轮 Agent 强化学习框架（AWM 论文训练所用的框架之一）。

在此基础上，你只在 **流程、工程化、部署、应用层** 做修改和优化。

**一句话边界：你改的是"怎么用、怎么部署、怎么管、怎么看"，不改"模型有多强"。** 本仓库不产生任何新的模型性能数字；所有性能数字都引用自 AWM 论文，并明确标注为论文报告值。

---

## 1. 已知信息（Phase 0 必须逐条重新核实，不得直接信任）

### 1.1 上游仓库
- `https://github.com/Snowflake-Labs/agent-world-model`
- `https://github.com/Agent-One-Lab/AgentFly`
- 可能需要：`https://github.com/meta-pytorch/OpenEnv`（AWM README 称其环境基础设施已并入 OpenEnv 的 `envs/agent_world_model_env`）
- 数据集：Hugging Face `Snowflake/AgentWorldModel-1K`
- 模型：Hugging Face `Snowflake/Arctic-AWM-4B`、`Snowflake/Arctic-AWM-8B`、`Snowflake/Arctic-AWM-14B`

### 1.2 AWM CLI（来自其 README，需对照源码核实）
- `awm gen {scenario, task, db, sample, spec, env, verifier, all}`
- `awm env {start, check, check_all, reset_db}`；MCP 端点默认 `http://localhost:8001/mcp`
- `awm agent`（单任务演示，需要先 `vllm serve Snowflake/Arctic-AWM-4B`）
- `awm verify --mode {sql, code}`
- `awm bench --mode {bfcl, tau2, mcp_universe}` ——**本任务严禁调用，见 R1**
- 合成流水线的 LLM 由环境变量配置：`AWM_SYN_LLM_PROVIDER`、`OPENAI_API_KEY`、`OPENAI_BASE_URL`、`AWM_SYN_OVERRIDE_MODEL`、`EMBEDDING_OPENAI_API_KEY`

### 1.3 论文（arXiv 2602.10090，ICML 2026）
- 基座：Qwen3 thinking 4B/8B/14B；算法：GRPO；框架：AgentFly + veRL
- 训练子集：526 个环境、3,315 个任务；奖励：逐步格式检查 + 任务级结果校验的混合奖励；训练与推理对齐的滑动窗口历史
- **官方训练配方在哪里尚未确认**：可能在 AgentFly、OpenEnv、AWM 仓库中，也可能未公开。

### 1.4 论文报告的评测结果（Phase 0 必须对照 PDF 结果表逐格核对）
| 基准 | 4B 基座 → AWM | 8B 基座 → AWM | 14B 基座 → AWM |
|---|---|---|---|
| BFCLv3 Overall | 54.92 → 64.50 | 53.83 → 65.94 | 61.25 → 70.18 |
| τ²-bench Pass@1 | 待核对 | 待核对 | 待核对 → 39.03 |
| MCP-Universe 成功率 | 待核对 | 6.70 → 11.17 | 待核对 |

### 1.5 许可证
- AgentFly：Apache-2.0（需核实）
- AWM 仓库：根目录未发现 LICENSE 文件（需核实 pyproject、README、其它位置）
- 数据集卡、模型卡、veRL、OpenEnv 的许可证：需核实

---

## 2. 硬性规则（违反任何一条都必须立即停止并报告）

**R1 禁止跑评测。**
- 不得执行 `awm bench`（任何 mode）。
- 不得运行 BFCLv3、τ²-bench、MCP-Universe 的评测 harness。
- 不得对多条任务批量运行 agent 后汇总成功率、通过率或任何比率。
- 唯一允许的是单条任务的链路演示：`awm agent` 与 `awm verify` 各执行一次，仅用于证明"链路打通"，其结果不得写成任何比率或效果结论。

**R2 禁止改变被评测对象。**
- 不得修改 `Snowflake/AgentWorldModel-1K` 官方数据；自行合成的环境必须放在 `data/synth/`，manifest 标记 `origin: local-synth`，与官方数据严格分开。
- 不得修改上游官方训练配置文件。
- 不得启动完整 RL 训练。只允许 `smoke` 配置（≤1.7B 模型、LoRA、≤5 step），其产物目录标记 `NO_RESULTS`，不得用于任何效果结论。
- 不得使用自训权重替代 Arctic-AWM 去引用论文数字。

**R3 数字纪律。**
- 所有模型性能数字只能来自 `results/registry.yaml`。每条都要有 `source`（arXiv 编号、版本号、表号、行与列）和 `verified` 字段。
- 应用层（LangGraph 智能体、网关、记忆、服务）**没有被任何基准评测过**。任何文档不得出现"准确率提升""可靠性提升 X%""成功率"之类的表述；应用层只描述机制和能力，例如"写操作需人工审批""超过 N 步自动终止"。
- 凡引用论文结果处，必须注明："论文报告值，由官方模型在官方评测 harness 上测得，不是本仓库应用层的测量结果。"
- 允许自测的数字仅限工程事实，例如测试用例数、接口数量、镜像大小、冷启动耗时。这类数字必须由脚本实测，并在文档中写明测量命令。

**R4 反幻觉。**
- 调用任何上游函数、CLI 参数、配置键之前，必须先读源码确认其存在，并在 `docs/RECON.md` 记录"文件路径:行号"。
- 找不到就停下报告，不得猜测，不得自造同名接口。
- vLLM 的 tool-call parser 与 chat template、LangGraph API、MCP Python SDK API 同样处理：以已安装版本的源码或官方文档为准，版本号写入 RECON.md。

**R5 上游隔离。**
- 上游仓库以 git submodule 形式放在 `third_party/`，固定到具体 commit SHA。
- 不得直接修改 submodule 内的文件。确需改动时，以 `patches/NNNN-<desc>.patch` 形式提交，并在 `docs/DECISIONS.md` 新增一条 ADR 说明原因。
- 自有代码全部放在 `src/workbench/`。
- `docs/UPSTREAM.md` 必须精确到文件级，写明哪些是上游、哪些是本仓库新增。

**R6 许可证。**
- Phase 0 必须查清 AWM 仓库、AgentWorldModel-1K、Arctic-AWM 模型卡、AgentFly、veRL、OpenEnv 的许可证，写入 UPSTREAM.md。
- 任何一项许可证缺失，或其条款与"公开 GitHub 仓库 + 个人作品展示"的用途冲突时，停止并报告。
- 数据集和模型权重不得提交进仓库，只提供下载脚本。

**R7 密钥。** 不得提交任何 key。提供 `.env.example`；pre-commit 中加入密钥扫描工具（gitleaks 或 detect-secrets，先确认能安装，二选一）。

**R8 诚实汇报。**
- 测试必须真实执行，报告中贴出命令和输出结尾；失败就如实报告失败。
- 不得用空实现或固定返回值伪装功能可用。未完成项写入 `docs/LIMITATIONS.md`。
- 需要 GPU 或外部 API、而本地未能验证的步骤，在代码注释和文档中标注 `UNVERIFIED-LOCAL`，并写明验证所需的资源。

**R9 提交纪律。**
- 每个子任务至少一个小提交，使用 Conventional Commits（feat / fix / docs / test / chore / refactor），提交信息写明"做了什么、为什么"。
- 每个提交都应能在 mock 模式下通过 `make test`。
- 不得 force push，不得改写历史，不得修改 git 作者配置。

**R10 运行环境。** 假设开发机没有 GPU。所有自有代码必须能在 CPU + mock LLM 下跑通测试；GPU 相关部分只写脚本和文档。

**R11 依赖隔离。**
- 应用层（app）与训练层（train）使用两个独立的 uv 环境。不得为了装进同一个环境而升级或降级上游固定的依赖。
- 应用层代码不得 import 任何训练依赖（torch 分布式、veRL、AgentFly 训练模块）。

**R12 阶段闸门。** 每个 Phase 结束时，按第 5 节格式输出阶段报告，然后**停下，等我回复"继续"**。不得跨阶段连续执行。

**R13 范围控制。** 不得新增本任务书未列出的大功能。更好的想法写进 `docs/IDEAS.md`，不要实现。

**R14 语言。**
- 文档用中文，README 开头附 5 行英文摘要。
- 代码标识符和注释用英文。
- 提交信息用英文。

**冲突处理：** 任务书与上游实际情况冲突时，以上游源码为准，停下报告，不得自行发挥。

---

## 3. 目标仓库结构

```
bizagent-workbench/
├── CLAUDE.md                     # Phase 0 生成：R1–R14 的要点摘要，供后续会话自动加载
├── README.md
├── Makefile
├── pyproject.toml                # app 环境
├── train/pyproject.toml          # train 环境（独立）
├── .env.example
├── .pre-commit-config.yaml
├── .github/workflows/ci.yml
├── third_party/
│   ├── agent-world-model/        # submodule，固定 SHA
│   ├── AgentFly/                 # submodule，固定 SHA
│   └── OpenEnv/                  # 仅当 Phase 0 确认需要时添加
├── patches/
├── configs/
│   ├── app.yaml
│   ├── tool_policy.yaml
│   ├── pricing.yaml              # 合成成本估算用的价格表
│   ├── serving/arctic-awm-4b.yaml
│   └── train/{paper_mirror.yaml, smoke.yaml}
├── scripts/                      # serve_vllm.sh、download_data.sh 等
├── src/workbench/
│   ├── config.py  doctor.py  cli.py
│   ├── llm/        # backends: vllm, openai_compat, mock_replay; client.py
│   ├── envs/       # manager.py, snapshot.py, ports.py, health.py, catalog.py
│   ├── gateway/    # server.py, policy.py, audit.py, ratelimit.py, errors.py
│   ├── agent/      # graph.py, state.py, nodes/, guards.py, memory.py, prompts/
│   ├── api/        # app.py, sse.py, schemas.py
│   ├── synth/      # runner.py, ledger.py, validate.py
│   ├── train/      # preflight.py, launch.py（只以子进程调用 train 环境）
│   ├── results/    # registry.py, check_numbers.py
│   └── obs/        # tracing.py, metrics.py
├── ui/                           # 静态 HTML/JS，无构建步骤
├── results/registry.yaml
├── data/                         # gitignored：awm1k/、synth/、runs/
├── docs/
│   ├── RECON.md  UPSTREAM.md  ARCHITECTURE.md  DECISIONS.md
│   ├── WALKTHROUGH.md  RESULTS.md  LIMITATIONS.md  IDEAS.md  CHANGELOG.md
└── tests/{unit, integration, fixtures}
```

---

## 4. 分阶段任务

### Phase 0 — 侦察（不写功能代码）

1. 初始化 git 仓库；添加 AWM 与 AgentFly 两个 submodule，固定到当前 HEAD，记录 SHA 与日期。
2. 通读两个上游，在 RECON.md 中写清以下内容，每条附"文件:行号"：
   - 入口点与 CLI 定义文件。
   - MCP server 实现，包括传输方式（streamable HTTP、SSE 还是 stdio）。
   - `awm agent` 的 agent loop 与 prompt。
   - `awm verify` 的两种模式。
   - `awm agent` 输出目录的结构。
   - 数据集的文件与字段。
   - 各自的 Python 版本与依赖要求。
3. **定位训练配方**：在 AgentFly、OpenEnv（`envs/agent_world_model_env`）、AWM 三处查找论文训练所用的配置、奖励实现与启动脚本。结论只能是以下三种之一：
   - (a) 找到官方配方（给出路径）；
   - (b) 只找到环境适配，没有完整配方；
   - (c) 未公开。

   不得根据论文文字自行"复刻"一个配方并称之为官方配方。
4. **核实论文数字**：打开 arXiv 2602.10090 的 PDF 结果表，逐格核对 §1.4。能读到的其它格子（τ²-bench 各领域、MCP-Universe 各类别、各尺寸的基座值）一并录入 `results/registry.yaml` 草稿，每格记录表号与论文版本。无法访问时标记 `verified: false`。
5. 完成 R6 的许可证调查。
6. 完成 R11 的依赖冲突分析：列出 AWM、AgentFly/veRL、vLLM、LangGraph、MCP SDK 的版本要求及冲突点。
7. 生成 CLAUDE.md：把 R1–R14 浓缩为不超过 40 行的要点。

**产出**：RECON.md、UPSTREAM.md 初稿、CLAUDE.md、`results/registry.yaml` 草稿。
**验收**：RECON.md 每条事实都有来源（文件:行号或 URL），并附"未能确认"清单。
**停止条件**：任一上游不可访问；AWM 许可证不允许本用途。训练配方结论为 (c) 时照常继续，但必须在报告中高亮。

### Phase 1 — 骨架与开发体验

1. uv 双环境：根目录为 app 环境；`train/` 为独立环境，本阶段只放 pyproject 与 lock，不安装 GPU 依赖。
2. `config.py`：用 pydantic-settings 读取 `.env` 与 `configs/app.yaml`。端口、路径、超时、并发上限、后端选择全部集中在这里，其它模块不得硬编码。
3. `workbench` CLI（typer）：`doctor`、`env`、`gateway`、`serve`、`agent`、`api`、`synth`、`train`、`results` 子命令先建空壳，均可 `--help`。
4. `workbench doctor` 检查以下各项，输出表格；有阻断项时返回非零退出码，GPU 缺失只算 warning：
   - Python 版本；
   - submodule 是否已初始化、SHA 是否匹配；
   - 数据是否已下载；
   - 端口占用；
   - GPU 是否可用；
   - 必要的环境变量；
   - 各 LLM 后端的连通性。
5. mock LLM 后端 `mock_replay`：从 `tests/fixtures/trajectories/*.jsonl` 按顺序回放消息，支持工具调用消息。夹具中的工具名必须取自 AWM 数据集里真实存在的场景，文件头注明"手写夹具，非模型输出"。
6. Makefile 目标：`setup`、`doctor`、`lint`（ruff + 对 `src/` 的 `mypy --strict`）、`test`（pytest，默认 mock）、`check-numbers`、`demo-mock`、`data`（下载 AWM-1K 到 `data/awm1k/`）。
7. pre-commit + GitHub Actions CI：lint + test，mock 模式，无 GPU。

**验收**：在一台无 GPU 的机器上新克隆仓库后，`make setup && make doctor && make lint && make test` 全部通过。

### Phase 2 — 环境管理层（Env Manager）

目标：把"手动 `awm env start` 一个场景"变成可被程序可靠调度的环境服务。

1. 优先调用 AWM 的 Python 接口；如果只能走 CLI，就用子进程封装，并写 ADR 说明原因。
2. **会话级隔离**：每个 session 启动时，把场景的初始 DB 复制到 `data/runs/<session_id>/`。提供 `snapshot`、`restore`、`diff`，其中 diff 比较表级行数与主键差异。
3. **生命周期**：
   - 端口池负责分配与回收端口。
   - 启动后按与 `awm env check` 等价的方式做健康检查，带超时。
   - 进程崩溃时标记为 unhealthy。
   - 提供 `workbench env up / down / ls / logs`。
4. **资源控制**：
   - 并发上限，超出时排队。
   - 空闲超时后自动回收。
   - 主进程退出时清理所有子进程（处理 SIGINT / SIGTERM）。
5. **场景目录**：从 `data/awm1k/` 建索引（场景名、工具数、任务数、表数），提供 `workbench env search <keyword>`。

**验收**：
- 单测覆盖端口分配、快照/恢复/diff、超时与崩溃处理（用假进程模拟）。
- 集成测试在 CPU 上真实启动 1 个 AWM 场景，并成功调用 `list_tools`。若因依赖问题无法完成，标记 UNVERIFIED-LOCAL 并说明原因。

### Phase 3 — MCP 网关（安全与治理）

目标：为智能体提供一个统一、安全、可审计的工具入口。

1. **路由**：网关本身是一个 MCP server，按 session 把请求路由到对应场景的 AWM MCP server。对外暴露 `list_tools` / `call_tool`，工具名加场景前缀以避免冲突。
2. **风险分级**：按工具名与描述中的动词启发式自动分为三级，`configs/tool_policy.yaml` 可对单个工具覆盖；分级结果可导出，供人工审核。
   - read：get / list / search / find / view；
   - write：create / update / add / set；
   - destructive：delete / cancel / refund / transfer / remove。
3. **策略**：
   - deny-first，session 级工具白名单。
   - read 直接放行。
   - write 与 destructive 需要由应用层审批流程签发的一次性审批令牌；具体哪些级别需要审批，可在配置中调整。
4. **审计**：每次调用写一行 JSONL，字段包括 trace_id、session、工具、参数摘要、结果摘要、耗时、策略决定、审批人。参数中疑似 PII（邮箱、手机号、卡号模式）在审计日志中脱敏。
5. **错误规范化**：
   - 上游报错统一转成结构化错误，并附可行动的提示，例如参数枚举的合法取值、缺失字段，便于模型自我纠正。
   - "空结果"与"错误"必须严格区分，禁止静默返回空。
6. **限流**：按 session × 工具的令牌桶限流。

**验收**：
- 单测覆盖分级、策略矩阵（三个级别 × 有无审批）、审计脱敏、限流。
- 集成测试：通过网关调用真实 AWM 场景中的一个 read 工具成功；一个 destructive 工具在无审批时被拒绝，且审计日志中留痕。

### Phase 4 — 模型服务层

1. **vLLM 部署**：`scripts/serve_vllm.sh` 与 `configs/serving/arctic-awm-4b.yaml`，写明模型名、端口、max-model-len、gpu-memory-utilization，以及工具调用相关参数。
   - tool-call parser 与 chat template 必须以 Arctic-AWM 模型卡和已安装 vLLM 版本的源码为准，并在 RECON.md 引证。
   - 无法确定时，沿用 AWM README 中的最简命令 `vllm serve Snowflake/Arctic-AWM-4B`，并在报告中说明。
2. **统一 LLM 客户端**：
   - 后端：vllm / openai_compat（DeepSeek 等）/ mock_replay。
   - 两层超时：连接、读取分相位超时，外加整体墙钟上限，以防流式响应"半开"挂死。
   - 指数退避重试，仅针对网络错误与 5xx。
   - token 用量记入 trace。
3. 切换后端只改配置，不改代码。

**验收**：单测用假服务器模拟慢响应与半开连接，覆盖超时、重试策略、token 记账。无 GPU 时，vLLM 实际启动标记 UNVERIFIED-LOCAL。

### Phase 5 — 应用层智能体（LangGraph）

> 重要：这一层没有被任何基准评测过。文档中只能描述机制，不能描述效果（R3）。

1. **图结构**：`intake → plan → act ⇄ observe → verify → approve（按需）→ respond`
   - **plan**：把用户请求拆成步骤清单，用结构化输出 + Pydantic 校验。校验失败时把错误信息回灌给模型重试，最多 2 次。
   - **act**：通过网关调用 MCP 工具。工具列表与描述在运行时从 `list_tools` 拉取，不得在 prompt 中手写副本。
   - **verify**：对照计划检查完成情况；遇到需审批的写操作前进入 approve。
   - **approve**：使用 LangGraph interrupt，等待 API 层审批。被拒绝则回到 plan 并附上拒绝原因。
2. **终止守卫**：以下每种终止原因单独记录：
   - 最大步数；
   - 同工具同参数的重复调用阈值；
   - 连续 N 步无状态变化；
   - token 预算；
   - 墙钟超时。
3. **记忆**：
   - 短期：checkpointer（SQLite）按 thread 持久化，支持中断后恢复。
   - 长期：用 Store 保存用户与业务事实，写入要过门槛。每条带来源标记（`user_stated` / `tool_result` / `inferred`），只有前两类可以写入；设置 TTL；提供查看与删除接口。
4. **Prompt 文件化**：放在 `src/workbench/agent/prompts/*.md`，带版本号；每次变更同步更新 `docs/CHANGELOG.md`。
5. **模型后端**：通过 Phase 4 的客户端调用，默认 mock，可切换到 vLLM（Arctic-AWM）或 DeepSeek。

**验收**：
- mock 下的端到端测试覆盖以下情形：
  - 正常完成；
  - 需要审批且被批准；
  - 被拒绝后重新规划；
  - 重复调用触发终止；
  - 校验失败后重试；
  - 中断后从 checkpoint 恢复；
  - 记忆写入门槛。
- 用真实 AWM 场景 + mock LLM 完整跑通一次流程。

### Phase 6 — 服务化与可观测

1. **FastAPI 端点**：
   - `POST /sessions`：选择场景，拉起隔离环境。
   - `POST /sessions/{id}/messages`：SSE 流式推送节点事件与工具调用。
   - `GET /sessions/{id}/trace`
   - `POST /approvals/{id}`
   - `GET /memory`、`DELETE /memory/{key}`
   - `GET /healthz`
   - `GET /metrics`：Prometheus 格式，包括请求延迟直方图、工具调用计数、错误计数、审批等待时长、活跃环境数。这些是运维指标，不是模型效果指标。
2. **连接与并发**：客户端断开即取消当前一轮；并发会话超过上限时返回 503。
3. **UI**（`ui/`，静态页面）：
   - 对话界面，实时显示执行步骤；
   - 工具调用时间线；
   - 审批面板；
   - 轨迹查看器：可加载 `awm agent` 的输出目录和本仓库的 trace，两者格式在 RECON 中确认后做适配；
   - DB diff 视图。
4. **Docker Compose**：app（api + gateway）、env-manager、vllm（profile: gpu，默认不启动）。

**验收**：
- API 测试（mock）覆盖所有端点与断连取消。
- `make demo-mock` 启动后，可在浏览器里完整演示一次"查询 → 写操作 → 审批 → 完成"。
- `docker compose config` 校验通过；是否能实际构建视本机 Docker 情况而定，未验证则标注。

### Phase 7 — 合成与训练的流程工程化（不改变任何实验设定）

1. **合成流水线编排**：`workbench synth run --scenarios N --out data/synth/<run_id>`
   - 按 AWM 的 gen 子步骤逐步执行，每步完成后写 checkpoint，支持断点续跑。
   - LLM 调用做缓存与重试。
   - 维护成本账本：每步的 token 数与费用估算，价格表放在 `configs/pricing.yaml`。
   - 结束后自动运行 `awm env check_all`，生成验证报告：能启动的环境数、工具数、失败原因分类。
   - 默认 dry-run，只打印计划；只有在用户提供 API key 并显式加 `--execute` 时才真正执行。
2. **训练启动器**（只以子进程方式调用 train 环境）：
   - `preflight`：检查 GPU 显存、CUDA / torch / vLLM / veRL 版本、数据路径、配置合法性。
   - `paper_mirror` profile：Phase 0 结论为 (a) 时，逐字段镜像官方配方，只用于阅读和资源估算，**任何参数下都不启动训练**；结论为 (b) 或 (c) 时不创建此 profile，并在文档中说明。
   - `smoke` profile：≤1.7B 模型 + LoRA、≤5 step、极小 batch，只用于演示"rollout → 奖励 → 更新"循环。产物目录标记 `NO_RESULTS`。
3. app 环境中不得 import 任何训练相关代码。

**验收**：dry-run 与 preflight 有单测（GPU 信息用 mock）；无 GPU 时 smoke 标记 UNVERIFIED-LOCAL。

### Phase 8 — 结果登记与文档交付

1. **数字守卫**：`results/registry.yaml` 定稿。`make check-numbers` 扫描 README 与 docs，凡是看起来像性能指标的数字（百分比、x.xx 分、Pass@k 等），只要不在 registry 或工程数字白名单里，就判为失败。
2. **RESULTS.md**：由 registry 自动生成，表头注明来源，并写明"论文报告值，非本仓库测量"；`verified: false` 的条目标注"待核对"。
3. **文档**：
   - **README**：一句话定位、架构图、3 分钟 mock 快速开始、GPU 部署路径、与上游的关系、结果说明（链接到 RESULTS.md）。
   - **ARCHITECTURE**：mermaid 分层图（环境层 / 网关 / 服务层 / 应用层 / 合成与训练），以及"一次带审批的写操作"的时序图。
   - **DECISIONS**：至少 10 条 ADR，每条包含背景、可选方案、决定、代价。至少覆盖以下问题：
     - 为什么用 submodule 而不是把上游代码拷进来；
     - 为什么拆成两个环境；
     - 为什么网关独立做成一个 MCP server；
     - 为什么 deny-first；
     - 为什么工具描述在运行时拉取；
     - 为什么记忆写入要按来源设门槛；
     - 为什么用两层超时；
     - 为什么区分空结果与错误；
     - 为什么不做效果评测；
     - 为什么合成的环境与官方数据隔离。
   - **WALKTHROUGH**：面向仓库主人的学习路线，按"环境 → 合成 → 服务 → 网关 → 智能体 → 训练循环"的顺序，每一步给出命令、预期输出，以及应该阅读的源文件（附路径）。
   - **UPSTREAM**：文件级的"上游 vs 本仓库"边界，以及各项许可证。
   - **LIMITATIONS**：所有 UNVERIFIED-LOCAL 项、训练配方的公开状态、应用层未经评测、合成环境未用于任何训练。
4. 按第 6 节完成最终自检。

---

## 5. 阶段报告格式（每个 Phase 结束时必须输出）

1. 完成项（对应任务编号）
2. 实际执行的命令，以及输出结尾（原样粘贴）
3. 新增 / 修改的文件清单
4. 本阶段核实过的上游事实（文件:行号）
5. 未完成或未验证的项目（UNVERIFIED-LOCAL），以及验证所需资源
6. 与任务书的偏差及原因（没有就写"无"）
7. 下一阶段计划
8. 需要我决定的问题（不超过 3 个）

输出完毕后停止，等我回复"继续"。

---

## 6. 最终验收清单（Phase 8 末尾逐条执行并贴出输出）

- [ ] `make setup && make doctor && make lint && make test && make check-numbers && make demo-mock` 全部通过
- [ ] `grep -rnE "awm bench|--mode (bfcl|tau2|mcp_universe)" src scripts Makefile` 无结果
- [ ] `results/registry.yaml` 每条都有 source；`verified: false` 的条目在 RESULTS.md 中标注"待核对"
- [ ] 各 submodule 固定在 RECON.md 记录的 SHA 上，工作区干净（对每个 submodule 执行 `git -C third_party/<name> status --porcelain`，输出为空）
- [ ] 仓库中无密钥（密钥扫描通过）；CI 通过
- [ ] `git log --oneline` 呈现为连续的小提交，每条信息都有意义
- [ ] LIMITATIONS.md 列出了全部未验证项

---

## 7. 停止条件汇总（遇到任一情况，立即停下报告，不要自行绕过）

- 上游仓库、数据集或模型不可访问
- 许可证缺失或与本用途冲突
- 需要的上游接口不存在，或与任务书描述不一致
- 某一步不可避免地要跑评测、改动官方数据 / 配置、或启动完整训练
- 依赖冲突只能通过修改上游固定版本来解决
- 任何要写进文档的性能数字在 registry 中找不到来源
