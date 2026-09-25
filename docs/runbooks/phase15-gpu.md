# Phase 15 GPU runbook：15A 推理链路与 15B smoke 训练

本文供仓库主人在自己租用的 GPU 机器上手动执行（那台机器上不运行 Claude Code，用户决定 D18、D21）。执行完成后，按 §5 把日志脱敏并回贴；Claude Code 据此更新 U1、U3、U4、U5、U8 与 `docs/LIMITATIONS.md`。

- 本文写于 2026-09-25，随分支 `phase9-verification` 提交并经 PR 合入 `main`。本轮没有执行（D22）；以后仓库主人在新分支上单独执行，回贴日志后另提 PR。命令、参数与预期输出都对照过源码，或在无 GPU 的容器里核对过（出处见各步骤，以及 `docs/RECON.md` 的"Phase 15 前置修复"与"Phase 15 runbook"两节）。**所有 GPU 步骤都没有实际执行过（UNVERIFIED-LOCAL）**：实际输出与本文不符时以实际为准，照实回贴，不要为了"通过"而改命令。
- 执行纪律：
  - 不做评测（R1）。每条调用模型的命令只执行 **1 次**、只跑 1 个任务；不重跑、不挑结果，也不汇总成比率（N2）。命令在调用模型之前就失败的（服务没起来、路径写错、依赖缺失），修好后可以再执行一次，回贴时写明。
  - 不调用任何付费 API（D14）。本文的模型调用全部指向本机 vLLM；机器上不需要、也不要设置任何 API key 或 token。
  - smoke 训练只记录三件事：循环能否跑完、峰值显存、耗时。产物标 `NO_RESULTS`，其中 reward 等指标不做任何解读（R2）。
  - `awm verify` 使用 `--mode code`。源码确认只有 sql 模式会调用 LLM 裁判（`awm/core/verify.py:416-433`），code 模式只执行数据集自带的纯代码 verifier，不需要裁判。本轮不执行 sql 模式；以后如果执行，裁判指向本机 vLLM，并在记录中注明"裁判为 Arctic-AWM-4B，仅用于打通链路"。
  - 不改 `third_party/` 下的任何文件（R5），不推送任何 Docker 镜像（D20）。回贴前先脱敏（N1，§5.2）。

## 0. 总览

| 段 | 机器 | 内容 | 对应待验证项 |
|---|---|---|---|
| §2 公共准备 | 两段都要做；同一台机器只做一次 | 取代码、uv 与 Python 3.12、应用环境、官方数据集、`workbench doctor` | — |
| §3 15A | 1 张 24 GB 显卡 | train 环境（不装 flash-attn）、下载 Arctic-AWM-4B、启动 vLLM、探针 1 次、`workbench agent run` 1 次、`awm agent` 1 次、`awm verify` 1 次 | U1、U8、U3（部分） |
| §4 15B | 1 张 A100 级显卡 | veRL 嵌套子模块、完整 train 环境（本机编译 flash-attn）、preflight、smoke 训练 1 次（3 step） | U3、U4、U5 |

一台 A100 可以依次做完 15A 与 15B：§2 只做一次，15A 结束时停掉 vLLM（§3.8），再进入 15B。

各步骤标题中的【外网：…】表示该步骤需要访问的外部站点，【本地】表示不需要外网。

## 1. 机器要求

| 项目 | 15A | 15B | 依据 |
|---|---|---|---|
| GPU | 1 张 24 GB 显卡，Ampere 或更新架构（RTX 3090/4090、A10、L4 等） | 1 张 A100（40 GB 或 80 GB；A800 同样可以） | Arctic-AWM-4B 权重约 8.8 GB（bf16，§3.2）；24 GB 卡按 `gpu_memory_utilization` 0.9 分配，扣除权重后留给 KV cache 的空间足以容纳一条 `max_model_len` 40960 的序列（估算）。preflight 要求至少 12000 MiB 可用显存（`src/workbench/train/preflight.py`） |
| 驱动 | `nvidia-smi` 右上角 `CUDA Version` ≥ 12.8 | 同左 | train 环境锁定 torch 2.10.0，其 CUDA 运行库为 12.8（`nvidia-cuda-runtime-cu12` 12.8.90），随 wheel 安装，不需要系统 CUDA。低于 12.8 时可能靠 CUDA 小版本兼容运行，但本仓库没有验证，建议换机器 |
| CUDA toolkit（`nvcc`） | 不需要 | 需要 12.x（flash-attn 要求 ≥ 11.7），并设置 `CUDA_HOME` | flash-attn 2.8.3.post1 只有源码包，而且没有对应 torch 2.10.0 的预编译 wheel，只能在本机编译（§4.2） |
| 内存 | ≥ 32 GB | ≥ 64 GB | flash-attn 编译时每个并行任务的峰值内存约 9 GB（flash-attn `setup.py` 中的注释），由 `MAX_JOBS` 控制 |
| CPU | ≥ 8 核 | ≥ 16 核更好 | 编译并行度 |
| 磁盘（数据盘可用空间） | ≥ 50 GB | ≥ 60 GB；与 15A 同机时 ≥ 70 GB | 估算：train 环境 10–15 GB（uv 缓存与环境在同一磁盘时以硬链接共享）、应用环境约 2 GB、Arctic-AWM-4B 约 9 GB、数据集约 0.5 GB；15B 另有 flash-attn 构建与 Qwen3-0.6B |
| 系统 | Linux x86_64，有 `gcc` | 同左 | `train/pyproject.toml` 只为 linux x86_64 锁定；Triton 在运行时用 `gcc` 编译小段代码 |
| Python | 3.12（uv 下载，或 conda 的 3.12） | 同左 | 根项目与 train 项目都要求 `>=3.12,<3.13` |

## 2. 公共准备（15A、15B 共用）

### 2.1 取代码【外网：GitHub】

```bash
nvidia-smi                       # 先看 GPU 型号、显存与 CUDA Version（≥ 12.8）
cd /root/autodl-tmp              # AutoDL 的数据盘；其他机器换成空间足够的目录
git clone --branch main https://github.com/IntheFesh/Agent2.git   # 或为 Phase 15 新开的分支；phase9-verification 合入 main 之前用它
cd Agent2
git submodule update --init third_party/agent-world-model third_party/AgentFly
git submodule status
```

- 预期：`git submodule status` 输出两行，SHA 分别以 `85e322f` 与 `1256586` 开头，行首是空格（`-` 表示未初始化，`+` 表示版本不符）。
- 判定：两行都符合即通过。
- 仓库为私有时，用你自己的凭据克隆；凭据不要出现在任何回贴内容里。
- GitHub 访问受限时有两个办法：
  - AutoDL 可以临时开启学术资源加速（`source /etc/network_turbo`，以 AutoDL 文档为准），下载结束后执行 `unset http_proxy https_proxy`；
  - 或者在能访问 GitHub 的机器上执行上面的 git 命令（做 15B 时再加上 §4.1 的命令），把整个 `Agent2` 目录打包上传后解压。
  不要使用来源不明的 GitHub 镜像站。

### 2.2 uv 与 Python 3.12【外网：PyPI；`uv python install` 需要 GitHub】

```bash
pip install uv==0.8.17           # 与本仓库验证时的版本一致；pip 使用机器上已配置的 PyPI 镜像
uv --version                     # 预期：uv 0.8.17
uv python find 3.12 || uv python install 3.12
```

- 判定：最后一条输出一个 Python 3.12 解释器的路径。
- 系统 Python 拒绝 `pip install`（报 `externally-managed-environment`）时，改用 `pipx install uv==0.8.17`。
- `uv python install` 需要从 GitHub 下载。下载失败时改用 conda：`conda create -y -p /root/autodl-tmp/py312 python=3.12`，然后在 §2.3 创建的 `env.sh` 末尾加一行 `export UV_PYTHON=/root/autodl-tmp/py312/bin/python3.12`。

### 2.3 公共环境变量与机器信息【本地】

在仓库根目录执行下面的命令，生成 `data/p15/env.sh`。**以后每开一个新终端，都先执行 `source data/p15/env.sh`**，它会切换到仓库根目录。`data/` 已被 git 忽略，本文的所有输出都写在 `data/p15/` 和 `data/train_runs/` 下。

```bash
mkdir -p data/p15/logs
cat > data/p15/env.sh <<'EOF'
# Phase 15 公共环境：每个新终端先 source 本文件
export REPO=$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)
export WORK=$(dirname "$REPO")                    # 仓库所在目录（数据盘）
export P15=$REPO/data/p15 LOG=$REPO/data/p15/logs
export PYTHONPYCACHEPREFIX=$REPO/.cache/pycache   # 不在 third_party/ 里留下 __pycache__
export UV_CACHE_DIR=$WORK/.cache/uv               # uv 缓存放数据盘
export UV_HTTP_TIMEOUT=300                        # 网络慢时下载大 wheel 不超时
export UV_NO_SYNC=1                               # uv run 不自动同步环境（原因见 §3.1）
export HF_HOME=$WORK/.cache/huggingface           # Hugging Face 缓存放数据盘
export HF_ENDPOINT=https://hf-mirror.com          # 国内镜像站；能直连 huggingface.co 时删掉这一行
export HF_HUB_DISABLE_XET=1                       # 不走 Xet 协议，全部经 HF_ENDPOINT 下载
export VLLM_NO_USAGE_STATS=1 DO_NOT_TRACK=1       # 关闭 vLLM 的使用统计上报
export NO_PROXY=localhost,127.0.0.1,::1 no_proxy=localhost,127.0.0.1,::1   # 本机服务不经代理
set -o pipefail
cd "$REPO"
# run <名字> <命令...>：命令、起止时间、全部输出与退出码写入 $LOG/<名字>.log，同时显示在屏幕上
run() { local name=$1; shift; { printf '$'; printf ' %q' "$@"; echo; echo "start $(date -u +%FT%TZ)"; "$@"; rc=$?; echo "end $(date -u +%FT%TZ) rc=$rc"; } 2>&1 | tee "$LOG/$name.log"; }
EOF
source data/p15/env.sh
run machine bash -c 'nvidia-smi; nvcc --version; gcc --version | head -1; uname -srm; nproc; free -g; df -h "$WORK"; uv --version; uv python find 3.12; git rev-parse HEAD; git submodule status'
```

- 预期：`$LOG/machine.log` 记录了 GPU、驱动、CUDA、内存、磁盘与当前提交；15A 机器没有 `nvcc` 属于正常情况。
- 判定：`nvidia-smi` 的 `CUDA Version` ≥ 12.8，磁盘满足 §1 的要求。
- 说明：
  - `HF_HUB_DISABLE_XET`：应用环境中的 huggingface_hub 1.32.0 装有 `hf_xet`，Xet 协议的下载不经过 `HF_ENDPOINT`（`huggingface_hub/constants.py:342`、`utils/_runtime.py:155-157`）。
  - `NO_PROXY`：如果开了代理，httpx 与 OpenAI SDK 默认会把发往本机服务（vLLM、环境服务）的请求也交给代理，这里显式排除本机地址。
  - `run` 在日志第一行记下完整命令，结束时打印 `rc=<退出码>`；`rc=0` 表示成功。判断成败以日志最后一行的 `rc` 为准（`run` 本身总是返回 0）。

### 2.4 应用环境【外网：PyPI（files.pythonhosted.org）】

```bash
run app-sync uv sync --frozen
```

- 预期：结尾是 `Installed N packages`，最后一行 `rc=0`。
- 判定：`rc=0`。
- `--frozen` 严格按 `uv.lock` 里记录的地址（files.pythonhosted.org）下载。对 `--frozen`，PyPI 镜像设置（`UV_DEFAULT_INDEX` 等）不起作用；改用 `--locked` 又会因为锁文件记录的是 pypi.org 而报"lockfile needs to be updated"（uv 0.8.17 实测，见 RECON）。下载太慢时，开启 AutoDL 学术资源加速或配置 HTTP 代理。**不要为了换镜像重新执行 `uv lock`**：那样得到的是一个没有验证过的环境。

### 2.5 官方数据集【外网：HF 镜像】

```bash
run data env AWM1K_REVISION=dde80a0283fe781bdc51656bce57063dc5650213 ./scripts/download_data.sh data/awm1k
```

- 预期：最后一行是 `dataset ready in data/awm1k (revision dde80a0283fe781bdc51656bce57063dc5650213)`，`data/awm1k` 下有 8 个 `gen_*.jsonl`，合计约 500 MB（`du -sh data/awm1k`）。
- 判定：`rc=0`。
- 数据集许可证为 CC-BY-4.0，只下载、不入库（R6）。

### 2.6 自检【本地】

```bash
run doctor uv run workbench doctor
```

- 预期：`python` 为 ok（3.12）；两个 `submodule:*` 为 ok（`pinned at 85e322f…` 与 `pinned at 1256586…`）；`dataset`、`ports`、`gpu`（`nvidia-smi found`）、`env-vars`（`backend=mock_replay`）、`llm`（`mock_replay fixture …`）都是 ok。
- 判定：没有 fail，`rc=0`。

## 3. 15A：vLLM 服务 Arctic-AWM-4B 与三条链路

### 3.1 train 环境中的推理部分【外网：PyPI】

```bash
run train-sync-15a bash -c 'cd train && uv sync --frozen --no-install-package flash-attn'
run train-probe uv run --project train python -c "import torch, vllm; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), vllm.__version__)"
run submodules-15a bash -c 'git -C third_party/AgentFly status --porcelain; git -C third_party/agent-world-model status --porcelain; echo checked'
```

- 预期：
  - 第一条下载约 5.4 GB（240 余个包，最大的是 torch、cudnn、cublas、vllm），`rc=0`；
  - 第二条输出 `2.10.0 12.8 True 0.19.0`（torch 版本号可能带 `+cu128` 后缀）；
  - 第三条只输出 `checked`。AgentFly 以 editable 方式安装，产生的 `*.egg-info`、`build/` 都在 AgentFly 自己的 `.gitignore` 里，不会改动子模块。
- 判定：三条都符合预期。
- 为什么这里不装 flash-attn：vLLM 在 CUDA 上使用自带的 `vllm.vllm_flash_attn`，外部 flash-attn 包只在 ROCm 上用到（vLLM 源码 `vllm/v1/attention/backends/fa_utils.py:18-44`），而 flash-attn 只能在本机编译（§4.2），15A 不需要它。这也是 `env.sh` 设置 `UV_NO_SYNC=1` 的原因：不带 `--no-sync` 的 `uv run --project train` 会先同步环境，从而开始编译 flash-attn。

### 3.2 下载模型【外网：HF 镜像】

```bash
run model-download uv run hf download Snowflake/Arctic-AWM-4B --revision 437dfa0e12702901eb41c30e9326a90996549650 --local-dir "$WORK/models/Arctic-AWM-4B"
run model-check bash -c 'ls -l "$WORK/models/Arctic-AWM-4B"; md5sum "$WORK/models/Arctic-AWM-4B/chat_template.jinja"'
```

- 预期：目录中有 `config.json`、`chat_template.jinja`、`model-00001-of-00002.safetensors`（约 5.0 GB）、`model-00002-of-00002.safetensors`（约 3.8 GB）、`model.safetensors.index.json` 与 tokenizer 文件；`chat_template.jinja` 的 md5 为 `da05f6b8a81932c7cf5f26eb545d4417`。
- 判定：两个权重分片都在，md5 一致（与 `docs/verification/2026-09-24-model-cards.md` 记录的相同）。
- 模型许可证 Apache-2.0；权重不入库（R6）。

### 3.3 启动 vLLM【本地】

vLLM 在前台运行，这一步放在**单独的终端**里做（先 `source data/p15/env.sh`）。启动参数全部来自 serving profile；这里只把 `model` 换成本地目录，`served_model_name` 保持 `Snowflake/Arctic-AWM-4B`。

```bash
sed "s#^model: Snowflake/Arctic-AWM-4B\$#model: $WORK/models/Arctic-AWM-4B#" configs/serving/arctic-awm-4b.yaml > "$P15/arctic-awm-4b.local.yaml"
run vllm-cmd uv run workbench serve vllm-cmd --profile "$P15/arctic-awm-4b.local.yaml"
PROFILE="$P15/arctic-awm-4b.local.yaml" scripts/serve_vllm.sh 2>&1 | tee "$LOG/vllm-serve.log"
```

- `vllm-cmd` 的预期输出（只有模型路径因机器而异）：
  `vllm serve /root/autodl-tmp/models/Arctic-AWM-4B --host 127.0.0.1 --port 8000 --served-model-name Snowflake/Arctic-AWM-4B --gpu-memory-utilization 0.9 --enable-auto-tool-choice --tool-call-parser hermes`
- 回到第一个终端，等待服务就绪（最多 15 分钟）：

```bash
run vllm-ready bash -c 'for i in $(seq 180); do curl -sf http://127.0.0.1:8000/v1/models && break; sleep 5; done; echo; grep -E "\"auto\" tool choice has been enabled|GPU KV cache size|Maximum concurrency for|Application startup complete" "$LOG/vllm-serve.log"; nvidia-smi --query-gpu=name,memory.used,memory.total --format=csv'
```

- 预期：
  - `/v1/models` 返回的 JSON 里 `id` 为 `Snowflake/Arctic-AWM-4B`，`max_model_len` 为 40960（取自模型 config）；
  - vLLM 日志中有 `"auto" tool choice has been enabled.`（`vllm/parser/parser_manager.py:202`）、`GPU KV cache size: … tokens`、`Maximum concurrency for 40,960 tokens per request: …`（`vllm/v1/core/kv_cache_utils.py:1319-1329`）和 uvicorn 的 `Application startup complete.`。
- 判定：以上都出现即通过。`/v1/models` 一直没有返回时，看 `$LOG/vllm-serve.log` 的报错（§6）。

### 3.4 探针：原生 tools 与文本协议各 1 次【本地】

```bash
run probe uv run workbench serve probe
```

`workbench serve probe`（`src/workbench/llm/probe.py`）向本机服务各发 1 个请求：

- `native`：workbench agent 行动时的请求，由 `vllm` 后端自己构造并发送（带原生 `tools`、流式、开启思考）。
- `text`：`awm agent` 的第一个请求，直接调用 AWM 自己的 `generate_response`，使用 AWM 的 system prompt 与它对 `localhost` 地址附加的 vLLM 参数，不带 `tools`。

两个请求的原始响应都会保存，以便区分服务返回的内容与客户端的解析结果。

- 预期：打印一个 JSON，`served_models` 为 `["Snowflake/Arctic-AWM-4B"]`，`native` 与 `text` 各有一个 `verdict`；`rc=0`（只有 `error` 或 `parser-interfered` 时 `rc=1`）。
- 判定（逐项记录，不因结果不理想而重跑）：

| 字段 | 取值与含义 |
|---|---|
| `native.verdict` | `native`：act 请求拿到了原生 `tool_calls`（U1 的主要待验证点）；`text-fallback`：服务没有给出原生调用，workbench 靠文本解析兜底，链路仍能工作；`no-call`：模型没有调用工具；`error`：请求失败，例如 HTTP 400 表示服务启动时没带 tool parser 参数 |
| `text.verdict` | `ok`：不带 `tools` 的请求里，`<tool_call>` 文本留在 `content` 中，AWM 自己的解析器能读出调用（D11 结论成立）；`parser-interfered`：parser 把调用从文本中取走了，与源码结论相反；`no-call`：模型没有按文本协议输出，以 §3.6 的实际运行为准 |
| `think_open` / `think_close` / `tool_call_text_in_content` / `content_tail` | 用来判断 `<tool_call>` 是否出现在思考内容里（U1 记录的另一个疑点：没有设置 reasoning parser） |

### 3.5 `workbench agent run`（vllm 后端，1 次）【本地】

与 Phase 12 相同的场景和任务（官方 `e_commerce_33` 任务 0）。

```bash
run wb-agent env WORKBENCH_LLM__BACKEND=vllm WORKBENCH_LLM__TOTAL_TIMEOUT_S=600 WORKBENCH_AGENT__WALL_CLOCK_S=1200 WORKBENCH_ENV__RUNS_DIR=data/p15/wb-runs \
  uv run workbench agent run --scenario e_commerce_33 --approve auto --approver p15-operator \
  "Search for 'wireless noise cancelling headphones', sort results by average customer rating, and add the top-rated item under \$200 to my cart in quantity 1."
```

- 两个时限的调整：单次调用上限 `llm.total_timeout_s` 默认 180 秒（ADR-008 的第二层超时），整次运行上限 `agent.wall_clock_s` 默认 300 秒，都没有针对单卡本地模型测过。本地 4B 模型开启思考、单次最多生成 8192 个 token 时可能超出，所以这一次运行放宽到 600 秒与 1200 秒。这只是工程时限，不改变模型与提示；其余参数都用默认值。
- 预期：
  - 第一行 `session <id>: 39 tools from http://127.0.0.1:181xx/mcp`；
  - 之后是 `tool … -> …` 行，写操作前有 `approval_required` / `approval_granted`；
  - 最后是 `answer …` 与数据库 diff 的 JSON，`rc=0`。
- 判定（链路是否打通，不评判任务结果）：
  - `rc=0`；
  - `data/p15/wb-runs/<session>/trace.jsonl` 里有 `purpose` 为 `plan` 和 `act` 的 `llm` 事件，并至少有一次工具调用；
  - 最后一条 `done` 事件中的 `termination` 如实记录（正常结束或触发了某个预算）。
  - 任务是否完成、diff 是否符合预期，只作为这一次运行的现象记录。

### 3.6 `awm agent`（1 次，`--mcp_url` 模式）【本地】

与 Phase 12 相同：环境由本仓库 env-manager 启动（显式传 `--db_path`、`--temp_server_path`、`--output_dir`），`awm agent` 只连 MCP 地址。原因见 `docs/verification/2026-09-24-llm-chain.md` §4.1：`--scenario` 自动起服有上游缺陷。

```bash
WORKBENCH_ENV__RUNS_DIR=data/p15/awm-runs uv run workbench env serve > "$LOG/env-serve.log" 2>&1 &
echo $! > "$P15/env-serve.pid"; sleep 5
run env-up uv run workbench env up e_commerce_33 --session-id p15awm
```

- 预期：`env-up` 输出 JSON，`state` 为 `healthy`，`url` 形如 `http://127.0.0.1:18100/mcp`，`tools` 列出 39 个工具。
- 如果 `url` 的端口不是 18100，把下面命令中的地址换成实际地址。

```bash
run awm-agent uv run awm agent --scenario e_commerce_33 --task_id 0 --tasks_path data/awm1k/gen_tasks.jsonl \
  --mcp_url http://127.0.0.1:18100/mcp --api_url http://localhost:8000/v1 --model Snowflake/Arctic-AWM-4B \
  --output_dir data/p15/awm-agent/e_commerce_33_task_0
run env-diff uv run workbench env diff p15awm
run env-down uv run workbench env down p15awm
kill "$(cat "$P15/env-serve.pid")"
```

- `--api_url` 必须写 `localhost`，不能写 `127.0.0.1`：AWM 只在地址含 `localhost` 与 `v1` 时附加 vLLM 专用参数（`add_generation_prompt`、`min_tokens` 16、`enable_thinking`），并以 `tool` 角色回传工具结果（`awm/core/agent.py:355-379,417`）。其余参数保持上游默认（`max_iterations` 30、`temperature` 1.0、`max_tokens` 2048）。
- 机器上不要设置 `OPENAI_API_KEY`：AWM 会用占位值 `EMPTY`（`awm/tools.py:429`），本机 vLLM 不校验 key。
- 预期：
  - 日志中有 `Resolved task from scenario=e_commerce_33, task_id=0: …` 和 `Loaded 39 tools from MCP server`；
  - 每轮输出 `Assistant (N chars): …` 与 `Tool calls: N`；
  - 结尾是 `Run outputs saved to: data/p15/awm-agent/e_commerce_33_task_0`，`rc=0`；输出目录里有 `trajectory.json`。
- 判定（U8 剩下的一半，只看链路）：
  - `rc=0`；
  - 第 1 轮 `Tool calls: 1` 且调用的是 `list_tools`：文本协议在启用了 hermes parser 的服务上可用；
  - 之后各轮的调用、最终回答和 `env diff`（`changed` 是否为 true）如实记录，不作为成败标准。
  - 与 Phase 12 一样，模型输出可能在某一轮不含 `<tool_call>`（例如思考内容用完了 2048 个 token），AWM 会把它当作最终回答并结束循环。这是一次运行中的现象，照实记录即可。

### 3.7 `awm verify`（1 次，code 模式，无 LLM 裁判）【本地】

```bash
run verify uv run workbench verify --input data/p15/awm-agent/e_commerce_33_task_0 --mode code \
  --init-db data/p15/awm-runs/p15awm/initial.db --final-db data/p15/awm-runs/p15awm/work.db
```

- `workbench verify`（ADR-025）在 code 模式下只给 `awm verify` 进程白名单环境变量，不给任何 key，默认使用 `data/awm1k/gen_verifier.pure_code.jsonl`。`--mcp_url` 模式的输出目录里没有数据库，所以要显式传会话的初始库与工作库（`awm/core/verify.py:41-43,367-368`）。
- 执行前已审阅将被 `exec` 的 verifier（`e_commerce_33` 任务 0 的纯代码 verifier）：共 174 行，只 import `re` 和 `sqlite3`，以只读 URI（`mode=ro`）打开两个数据库，没有写 SQL，也没有文件、网络或子进程调用。
- 预期：打印 JSON，含 `"mode": "code"`、`verifier`、`output`（`…/verify.code.json`）与 `reward_type`；`rc=0`。
- 判定：`rc=0`，并生成了 `verify.code.json`。`reward_type` 的取值只记录、不解读（N2）。

### 3.8 收尾【本地】

```bash
run cleanup-15a bash -c 'find data/awm1k -newer data/awm1k/MANIFEST.json -type f | head; git -C third_party/AgentFly status --porcelain; git -C third_party/agent-world-model status --porcelain; ls data/p15/awm-agent/e_commerce_33_task_0; echo checked'
```

- 预期：只输出输出目录的文件列表（`trajectory.json`、`verify.code.json`、`verify.code.log`）和 `checked`：官方数据目录里没有新文件，两个子模块都没有改动。
- 之后在 vLLM 所在终端按 Ctrl-C 停止服务，并用 `nvidia-smi` 确认显存已经释放（接着做 15B 时必须先停掉 vLLM，否则 preflight 的显存检查不通过）。
- 按 §5 整理 15A 的回贴内容。

## 4. 15B：train 环境、preflight 与 smoke 训练

从新终端开始时，先 `source data/p15/env.sh`。如果是一台新机器，先完成 §2。

### 4.1 veRL 嵌套子模块【外网：GitHub】

```bash
run verl-submodule git -C third_party/AgentFly -c url."https://github.com/".insteadOf="git@github.com:" submodule update --init verl
run verl-status bash -c 'git -C third_party/AgentFly submodule status verl; cat third_party/AgentFly/verl/verl/version/version; git -C third_party/AgentFly status --porcelain; echo checked'
```

- AgentFly 的 `.gitmodules` 用的是 SSH 地址（`git@github.com:Agent-One-Lab/verl.git`）。`-c url…insteadOf` 只让这一次克隆改走 HTTPS，不修改 `.gitmodules`（TASK_v2 Phase 15 的要求）。
- 预期：`verl-submodule.log` 中有 `Submodule path 'verl': checked out '001f000ae2e4cf05bb94c01427898cbe68961141'`；`submodule status` 输出 ` 001f000ae2e4cf05bb94c01427898cbe68961141 verl (…)`，行首是空格；version 文件内容为 `0.8.0.dev`；AgentFly 的 `status --porcelain` 没有输出。 <!-- pragma: allowlist secret (a git commit SHA quoted from git's output) -->
- 这两条命令已在本仓库的开发容器中，对 AgentFly @`1256586` 的本地克隆原样执行过，结果与上面的预期一致（RECON "Phase 15 runbook"）。
- 判定：以上都符合（U4 的前半部分）。

### 4.2 完整安装 train 环境（本机编译 flash-attn）【外网：PyPI】

```bash
nvcc --version                       # 预期 release 12.x
cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes   # 容器的实际内存上限（字节；输出 max 表示没有限制，按租用页面标明的内存计算）
export CUDA_HOME=/usr/local/cuda     # 如果 nvcc 不在这里，改成 `which nvcc` 的上两级目录
export FLASH_ATTN_CUDA_ARCHS=80      # 只为 A100/A800（sm_80）编译；H100/H800 改成 90
export FLASH_ATTENTION_FORCE_BUILD=TRUE   # 直接编译，不去 GitHub 找预编译 wheel（torch 2.10.0 没有，已确认 404）
export MAX_JOBS=6                    # 取 min(CPU 核数/2, 容器内存GB/9)，按上面查到的内存上限计算
export NVCC_THREADS=4
run train-sync-15b bash -c 'cd train && uv sync --frozen'
run flash-attn-check uv run --project train python -c "import torch, flash_attn; from flash_attn import flash_attn_func; print(torch.__version__, flash_attn.__version__)"
```

- `MAX_JOBS` 必须手动设置：flash-attn 自动计算时用 `psutil` 读到的是宿主机内存，而不是容器的内存上限（flash-attn `setup.py:513-526`），在容器里可能导致编译被 OOM kill。
- `train/pyproject.toml` 已把 flash-attn 构建环境里的 torch 固定为锁定版本（`match-runtime`，ADR-027）。不固定时，uv 会在构建环境里装 PyPI 上最新的 torch（2026-09-25 为 2.14.0，CUDA 13），编出的扩展与运行时的 torch 2.10.0 不兼容。
- 预期：
  - 第一条会编译很久，日志中出现 `Building flash-attn==2.8.3.post1` 与 `Built flash-attn==2.8.3.post1`，`rc=0`；
  - 第二条输出 `2.10.0 2.8.3.post1`（torch 版本号可能带后缀）。
- 判定：两条都 `rc=0`，第二条能导入 `flash_attn_func`（U3）。
- 编译失败时，保留完整的 `$LOG/train-sync-15b.log`，回贴最后 200 行。

### 4.3 下载 Qwen3-0.6B【外网：HF 镜像】

smoke profile 通过 Hugging Face id `Qwen/Qwen3-0.6B` 引用模型。这里先把模型下载到 `$HF_HOME` 缓存，训练时再用 `HF_HUB_OFFLINE=1` 只读缓存。

```bash
run qwen-download uv run hf download Qwen/Qwen3-0.6B
run qwen-snapshot ls "$HF_HOME/hub/models--Qwen--Qwen3-0.6B/snapshots/"
```

- 预期：快照目录名就是所用的 revision；2026-09-25 查询时 `main` 为 `c1899de289a04d12100db370d81485cdf75e47ca`。不一致时照实记录。
- 下载时不要加 `--revision <哈希>`：那样不会写入 `refs/main`，离线模式下按 id 加载会找不到缓存。

### 4.4 preflight【本地】

```bash
run preflight env HF_HUB_OFFLINE=1 uv run workbench train preflight
```

- 预期：表格中 7 项都是 ok：
  - `profile`：`smoke: Qwen/Qwen3-0.6B (0.6B), LoRA, 3 steps, NO_RESULTS`；
  - `gpu`：`1 GPU(s) [NVIDIA A100…], max free … MiB (need 12000)`；
  - `train-env`：`torch 2.10.0… (cuda 12.8), vllm 0.19.0, cuda_available=True`；
  - `verl`：`verl 0.8.0.dev`；
  - `config-keys`：`… override keys exist in the pinned config`；
  - `hydra-compose`：`Hydra composed ppo_trainer with all overrides`；
  - `data`：`8 rows in configs/train/smoke_data.json`。
- 判定：`rc=0`，没有 fail（U4 的后半部分：Hydra 组合在 train 环境中通过）。
- 所有探测都在训练将要使用的白名单环境里运行（ADR-026）。

### 4.5 查看将要执行的命令（dry run）【本地】

```bash
run launch-dry env HF_HUB_OFFLINE=1 uv run workbench train launch
```

- 预期：一个 `"mode": "dry-run"` 的 JSON，包含训练命令（`uv run --project train --no-sync python -m agentfly.cli train …`）、`env.passed` / `env.withheld`（只有变量名，没有值）与 preflight 结果。
- 判定：`env.passed` 中有 `PATH`、`HOME`、`CUDA_HOME`、`HF_HOME`、`HF_HUB_OFFLINE`、`VLLM_NO_USAGE_STATS`、`PYTHONPYCACHEPREFIX`、`UV_CACHE_DIR` 等；`env.withheld` 中只有与训练无关的变量或凭据类变量。
- 机器确实需要某个被拦下的变量时，用 `WORKBENCH_TRAIN__ENV_PASSTHROUGH='["变量名"]'` 放行，并在回贴中写明。

### 4.6 smoke 训练（1 次，3 step，NO_RESULTS）【本地】

```bash
nvidia-smi --query-gpu=timestamp,memory.used,memory.total --format=csv,noheader -l 1 > "$LOG/15b-gpu-mem.csv" &
echo $! > "$P15/gpumon.pid"
run train-launch env HF_HUB_OFFLINE=1 uv run workbench train launch --execute
kill "$(cat "$P15/gpumon.pid")"
awk -F', ' '{gsub(/ MiB/, "", $2); if ($2 + 0 > m) m = $2 + 0} END {print "peak memory.used:", m, "MiB"}' "$LOG/15b-gpu-mem.csv" | tee "$LOG/15b-gpu-peak.txt"
export RUN_DIR=$(ls -td data/train_runs/*_smoke | head -1)
run train-result bash -c 'echo "$RUN_DIR"; ls "$RUN_DIR"; head -3 "$RUN_DIR/NO_RESULTS"; grep -c "^step:" "$RUN_DIR/train.log"; grep -o "^step:[0-9]*" "$RUN_DIR/train.log"; tail -n 5 "$RUN_DIR/train.log"'
```

- 训练输出写在 `$RUN_DIR/train.log`，屏幕上只有 `workbench` 打印的结果 JSON：`{"mode": "execute", "run_dir": "...", "returncode": N}`。
- 预期：
  - `returncode` 为 0；
  - `$RUN_DIR` 下有 `NO_RESULTS`、`command.json`、`env_names.json`、`train.log`；
  - `train.log` 中有 3 行以 `step:1`、`step:2`、`step:3` 开头的指标行（veRL console logger 的格式为 `step:N - key:value - …`，`verl/utils/logger/aggregate_logger.py:26-31`）。
- 判定（U5）：`returncode` 为 0，且出现了 3 个 step 行，即"rollout → reward → update"循环跑通。
- 需要记录的只有三项：是否跑通；峰值显存（`15b-gpu-peak.txt`，是整卡已用显存的峰值，包含 vLLM rollout 按 `gpu_memory_utilization` 0.4 预留的部分）；耗时（`train-launch.log` 的 start 与 end 时间差）。
- step 行里的 reward、score 等数值只作为日志原样保存，不解读，也不写进任何结论（R2）。

## 5. 回贴清单与脱敏

### 5.1 生成回贴文件

15A 结束后执行：

```bash
{
  for f in machine app-sync data doctor train-sync-15a train-probe submodules-15a model-download model-check vllm-cmd vllm-ready probe wb-agent env-up awm-agent env-diff env-down verify cleanup-15a; do
    echo "===== $f.log"; tail -n 150 "$LOG/$f.log"
  done
  echo "===== vllm-serve.log（前 60 行）"; head -n 60 "$LOG/vllm-serve.log"
  echo "===== vllm-serve.log（关键行与报错）"
  grep -nE '"auto" tool choice|GPU KV cache size|Maximum concurrency|Application startup complete|ERROR|Error|Traceback' "$LOG/vllm-serve.log" | head -n 60
  echo "===== workbench agent trace"; cat data/p15/wb-runs/*/trace.jsonl
  echo "===== verify.code.json"; cat data/p15/awm-agent/e_commerce_33_task_0/verify.code.json
  echo "===== awm agent trajectory（摘要）"
  uv run python - <<'PY'
import json
from pathlib import Path
t = json.loads(Path("data/p15/awm-agent/e_commerce_33_task_0/trajectory.json").read_text())
print({k: t.get(k) for k in ("scenario", "task_id", "model", "api_url", "max_iterations", "temperature", "total_iterations")})
for s in t["trajectory"]:
    calls = [(c.get("name"), c.get("arguments")) for c in s.get("tool_calls") or []]
    resp = (s.get("tool_response") or {}).get("content") or ""
    print(f"#{s.get('iteration')} final={bool(s.get('is_final'))} calls={json.dumps(calls, ensure_ascii=False)[:300]}")
    print("   content(last 400):", json.dumps((s.get("content") or "")[-400:], ensure_ascii=False))
    print("   response(first 300):", json.dumps(resp[:300], ensure_ascii=False))
PY
} > "$P15/paste-15a.txt" 2>&1
wc -c "$P15/paste-15a.txt"
```

15B 结束后执行：

```bash
RUN_DIR=${RUN_DIR:-$(ls -td data/train_runs/*_smoke | head -1)}
{
  for f in machine verl-submodule verl-status train-sync-15b flash-attn-check qwen-download qwen-snapshot preflight launch-dry train-launch train-result; do
    echo "===== $f.log"; tail -n 200 "$LOG/$f.log"
  done
  echo "===== 15b-gpu-peak.txt"; cat "$LOG/15b-gpu-peak.txt"
  echo "===== train.log（最后 200 行）"; tail -n 200 "$RUN_DIR/train.log"
} > "$P15/paste-15b.txt" 2>&1
wc -c "$P15/paste-15b.txt"
```

两段在同一台机器上执行时，15B 的 `machine.log` 与 15A 相同，照样附上即可。

### 5.2 脱敏（N1，必须在回贴前完成）

```bash
uv run python scripts/redact_paste.py "$P15/paste-15a.txt"     # 生成 paste-15a.redacted.txt，并打印各类替换次数
uv run python scripts/redact_paste.py "$P15/paste-15b.txt"
grep -nEi 'api[_-]?key|token|secret|passw|bearer|@' "$P15"/paste-15?.redacted.txt | grep -vE 'max_tokens|prompt_tokens|completion_tokens|total_tokens|tokens_used|token_budget|min_tokens|tokenizer|TOKENIZERS_PARALLELISM|tokens per request|git@github.com' | head -n 40
```

- `scripts/redact_paste.py` 替换的内容：
  - 形如 `hf_…`、`sk-…`、`ghp_…` 的 token；
  - URL 中的用户名与密码；
  - `Bearer` 后面的值；
  - `XXX_KEY=`、`XXX_TOKEN=` 之类赋值中的值；
  - 邮箱、卡号与电话号码（沿用网关审计日志的规则，ADR-016；电话在此基础上排除了日志中常见的几类数字，见下）。
  电话按数据集的写法识别：带 `-`、空格、括号或 `+` 的号码，以及大陆手机号。
  以下内容会保留，因为它们只是看起来像电话或密钥：版本号与 IPv4 地址（例如驱动版本）、小数、按列对齐的表格数字、单独的整数（文件大小、计数）、只写了变量名的赋值（例如 `API_KEY_ENV=DEEPSEEK_API_KEY`），以及 `git@github.com` 这类 SSH 地址。
- 最后一条 grep 列出剩下的可疑行。逐行确认：只剩变量名（例如 `env_names.json` 里的 `HF_TOKEN`）没有问题；看到任何真实的值，手动改成 `[REDACTED]`。
- 数据集里的合成用户资料（姓名、地址、电话等）也按上面的规则替换，不因为是合成数据就保留。
- 回贴 `paste-15a.redacted.txt` 与 `paste-15b.redacted.txt` 的全文；太长就分几次贴，并注明顺序。不要回贴原始的 `paste-15?.txt`。
- 另外请用几句话写明：机器型号与租用平台、是否开启了学术加速或代理、有没有命令重试过（原因是什么），以及与本文不一致的地方。

## 6. 常见问题

| 现象 | 处理 |
|---|---|
| `uv sync` 下载很慢或超时 | 开启 AutoDL 学术资源加速或 HTTP 代理后重试；不要改 `uv.lock`（§2.4） |
| vLLM 报 `max seq len (40960) is larger than the maximum number of tokens that can be stored in KV cache` | 显存不足以容纳一条最长序列。换 24 GB 以上的显卡；不要改 profile 中的 `max_model_len`（那样验证的就不是当前 profile），照实回贴 |
| vLLM 或 Triton 报 `Python.h` 或 `gcc` 相关的编译错误 | 安装 `gcc`；使用 uv 下载的 Python 3.12 或 conda 的 Python（都自带头文件） |
| 探针的 `native.verdict` 为 `error` 且含 HTTP 400 | 服务没有带 `--enable-auto-tool-choice --tool-call-parser hermes` 启动。检查 `vllm-cmd.log` 是否与 §3.3 一致 |
| `env up` 报端口被占用 | 先确认 §3.5 已经结束，再 `uv run workbench env ls` 查看残留的会话，`uv run workbench env down <id>` 后重试 |
| flash-attn 编译进程被 kill（`Killed` 或 `signal 9`） | 调小 `MAX_JOBS`（例如 2）后重跑 §4.2；编译不调用模型，不受 N2 限制 |
| preflight 的 `gpu` 为 fail | 15A 的 vLLM 还在占用显存，停掉后重试 |
| 训练报找不到 `Qwen/Qwen3-0.6B` | 确认 §4.3 下载时没有加 `--revision`，并且 `HF_HOME` 与下载时相同 |

## 7. 预计耗时（估计值，不是测量值）

| 步骤 | 预计耗时 | 主要取决于 |
|---|---|---|
| §2 公共准备 | 20–40 分钟 | GitHub 与 PyPI 的访问速度 |
| §3.1 train 环境（推理部分） | 10–40 分钟 | 约 5.4 GB 下载 |
| §3.2 下载模型 | 5–30 分钟 | 约 9 GB，镜像站速度 |
| §3.3 启动 vLLM | 3–10 分钟 | 加载权重、编译与 CUDA graph 捕获 |
| §3.4–§3.7 探针与三条链路 | 20–60 分钟 | 本地模型的生成速度与思考长度 |
| **15A 合计** | **约 1–3 小时** | |
| §4.1–§4.3 子模块、Qwen3-0.6B | 5–15 分钟 | 网络 |
| §4.2 编译 flash-attn | 30–90 分钟 | CPU 核数与 `MAX_JOBS` |
| §4.4–§4.6 preflight 与 smoke 训练 | 15–45 分钟 | Ray、vLLM rollout 与 FSDP 初始化 |
| **15B 合计** | **约 1–2.5 小时** | |
