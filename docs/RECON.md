# RECON — Phase 0 侦察报告

> 本文件记录 Phase 0 对上游的逐条核实结果。规则（R4）：每条事实都附"文件:行号"或固定到 commit SHA 的 URL；
> 找不到的内容列入文末"未能确认"清单，不做猜测。
> 本文件不包含任何模型性能数字；论文数字只登记在 `results/registry.yaml`（R3）。

## 0. 元信息

| 项 | 值 |
|---|---|
| 侦察日期 | 2026-09-24 |
| 执行环境 | Linux 容器，无 GPU（`nvidia-smi` 不存在）；默认 `python3` 为 3.11.15，另有 `/usr/bin/python3.12`（3.12.3）；uv 0.8.17；git 2.43.0 |
| 网络限制 | 本环境出口策略拒绝 `huggingface.co` 与 `arxiv.org`：`curl` 返回 `CONNECT tunnel failed, response 403`；WebFetch 返回 `EGRESS_BLOCKED`。GitHub（git 匿名读）与 PyPI 可达。 |
| 行号引用约定 | 子模块内文件写相对仓库根的路径（如 `third_party/agent-world-model/awm/cli.py:113`）；未纳入子模块的上游写固定 SHA 的永久链接；PyPI 包写 wheel 名 + 包内路径 |

### 0.1 上游版本固定

| 上游 | URL | 固定 SHA | 提交日期 | 纳入方式 |
|---|---|---|---|---|
| AWM | https://github.com/Snowflake-Labs/agent-world-model | `85e322f69279e3b3325b7377ec3bab788514e9cb` | 2026-05-28 | submodule `third_party/agent-world-model`（shallow） |
| AgentFly | https://github.com/Agent-One-Lab/AgentFly | `1256586b1109ba8e0dc0f179f8515b4567d09df4` | 2026-05-06 | submodule `third_party/AgentFly`（shallow） |
| └ AgentFly 嵌套 veRL fork | https://github.com/Agent-One-Lab/verl | `001f000ae2e4cf05bb94c01427898cbe68961141` | 2026-05-05 | AgentFly 的嵌套 submodule，**未初始化**；仅在 scratch 中只读检索 |
| └ AWM 嵌套评测 harness | https://github.com/Raibows/mcp-adapted-bench | `2dff8bdfc35082e5b6980c4954b6074159a15854` | 2026-05-22 | AWM 的嵌套 submodule，**未初始化**（R1）；仅在 scratch 中只读检索，未执行任何代码 |
| OpenEnv | https://github.com/meta-pytorch/OpenEnv | `e401886d23aab1be92493ea15e5d7e2cdf7e657b` | 2026-09-23 | **未纳入**（见 §3.4 结论），仅在仓库外克隆只读侦察 |

核实命令：`git submodule status`；`git -C third_party/<name> log -1 --format='%H %cI %s'`；`git ls-tree HEAD <path>`（嵌套 gitlink）。

---

## 1. AWM（Snowflake-Labs/agent-world-model @ 85e322f）

以下路径均相对 `third_party/agent-world-model/`。

### 1.1 入口点与 CLI 定义

- 控制台脚本：`pyproject.toml:39-40` → `awm = "awm.cli:main"`。
- CLI 解析库：`simpleArgParser.parse_args_with_commands`（`awm/cli.py:3`，`awm/cli.py:113-126`）。
- 命令枚举：`TopCmd`（`awm/cli.py:6-16`）、`GenCmd`（`awm/cli.py:19-35`）、`EnvCmd`（`awm/cli.py:38-46`）。
- 分发表：`DISPATCH`（`awm/cli.py:94-110`），每个命令对应模块的 `run(config)`（`awm/cli.py:125-126`）；每个模块用一个 `@dataclass Config` 定义参数。

| 命令 | 模块 | Config 定义 | 关键参数（默认值） |
|---|---|---|---|
| `gen scenario` | `awm/core/scenario.py` | `:26-48` | `input_path`, `output_path`, `target_count=1000`, `resume=False` |
| `gen task` | `awm/core/task.py` | `:11-19` | `input`, `output`, `num_tasks=10`, `limit=None` |
| `gen db` | `awm/core/db.py` | `:10-18` | `input`, `output`, `database_dir='./outputs/databases'` |
| `gen sample` | `awm/core/sample.py` | `:11-20` | `input_task`, `input_db`, `output`, `database_dir` |
| `gen spec` | `awm/core/spec.py` | `:10-15` | `input_task`, `input_db`, `output` |
| `gen env` | `awm/core/env.py` | `:20-28` | `input_spec`, `input_db`, `output`, `database_dir`, `max_retry=4` |
| `gen verifier` | `awm/core/verifier.py` | `:25-36` | `input_task`, `output`, `mode=sql`, `database_dir` |
| `gen all` | `awm/core/pipeline.py` | `:14-27` | `input='./outputs/seed_scenario.jsonl'`, `output_dir='outputs'`, `verifier_mode=sql` |
| `env start` | `awm/core/server.py` | `:13-26` | `scenario`, `envs_load_path`, `db_path=None`, `host='127.0.0.1'`, `port=8001`, `temp_server_path=None`, `output_dir=None` |
| `env check` | `awm/core/check.py` | `:7-12` | `url='http://localhost:8001/mcp'`, `timeout=10.0` |
| `env check_all` | `awm/core/test_env.py` | `:8-13` | **`input`**, `allowed_scenarios=None` |
| `env reset_db` | `awm/core/reset.py` | `:11-16` | `input_db`, `input_sample`, `database_dir`, `scenarios=None` |
| `agent` | `awm/core/agent.py` | `:49-80` | 见 §1.3 |
| `verify` | `awm/core/verify.py` | `:36-49` | 见 §1.4 |
| `bench`（R1 禁用） | `awm/eval/infer_eval.py` | 来自 `mcp_adapted_bench.common.infer_config`（`awm/eval/infer_eval.py:31-34`） | mode 枚举 `tau2/bfcl/mcp_universe`（[infer_config.py:8-11](https://github.com/Raibows/mcp-adapted-bench/blob/2dff8bdfc35082e5b6980c4954b6074159a15854/mcp_adapted_bench/common/infer_config.py#L8-L11)，只读查看） |

与任务书 §1.2 的对照：

- `gen` 8 个子命令、`env` 4 个子命令、`agent`、`verify --mode {sql,code}`（`awm/core/verify.py:32-34,45`）均与任务书一致。
- **不一致 1**：README 写 `awm env check_all --output outputs/gen_envs.jsonl`（`README.md:201`），源码参数名是 `input`（`awm/core/test_env.py:11`）。按冲突处理规则以源码为准。
- `bench` 的导入失败会被吞掉并登记为 `None`（`awm/cli.py:65-68,89`）。未初始化评测子模块时，`simpleArgParser` 如何处理 `None` 尚未验证（见 §9）。
- 合成 LLM 的环境变量（任务书列出的 5 个均存在，另有 3 个任务书未列出）：
  - `AWM_SYN_LLM_PROVIDER`：`awm/gpt.py:38-40`，`awm/tools.py:407`
  - `OPENAI_API_KEY` / `OPENAI_BASE_URL`：`awm/gpt.py:54-55`，`awm/tools.py:420-429`；默认 base URL 为 `https://api.openai.com/v1`（`awm/gpt.py:54`）
  - `AWM_SYN_OVERRIDE_MODEL`：`awm/gpt.py:42-46`；各 gen 步骤的 `pre_process` 会断言它已设置，例如 `awm/core/env.py:37-40`
  - `EMBEDDING_OPENAI_API_KEY`（`gen scenario` 必需）：`awm/core/scenario.py:63`
  - 任务书未列：`AZURE_ENDPOINT_URL`、`AZURE_OPENAI_API_KEY`（`awm/gpt.py:51-52`），`EMBEDDING_OPENAI_BASE_URL`（可选，`awm/core/scenario.py:64-65,85`）

### 1.2 MCP server 实现与传输方式

- 调用链：`awm env start` → `awm/core/server.py:198-199` `run` → `run_server`（`awm/core/server.py:146-171`）。
- 服务端代码不是手写的：从 `gen_envs.jsonl` 取该场景的 `full_code`（`awm/core/server.py:97-101`），然后
  - 把 `create_engine(...)` 那一行替换为指向会话 SQLite 文件的连接（`awm/core/server.py:105-109`）；
  - 在 `uvicorn.run(app` 之前注入 `FastApiMCP(app)` + `mcp.mount_http()`（`awm/core/server.py:111-128`，注入语句在 `:120-122`）；
  - `HOST` / `PORT` 可被环境变量覆盖（`awm/core/server.py:113-116`）。
- **传输方式：MCP Streamable HTTP**（不是 SSE，也不是 stdio）。证据：
  - fastapi-mcp 0.4.0 的 `mount_http()`（`fastapi_mcp-0.4.0-py3-none-any.whl:fastapi_mcp/server.py:312`）默认挂载路径为 `"/mcp"`（同文件 `:332`），它用 `FastApiHttpSessionManager` 包装 MCP SDK 的 `StreamableHTTPSessionManager`（同文件 `:351`；`fastapi_mcp/transport/http.py:52-56`）；
  - `json_response=True`（`fastapi_mcp/transport/http.py:21`），`stateless=False`（`fastapi_mcp/transport/http.py:56`）；
  - AWM 自己的客户端也以 `transport='streamable_http'` 连接（`awm/tools.py:159-162`，`awm/core/agent.py:289-292`）。
- 工具即 FastAPI 路由，工具名取 OpenAPI `operationId`（`fastapi_mcp/server.py:619`）。生成 prompt 要求每个路由都有 snake_case 的 `operation_id`（`awm/prompts.py:274`）。
- 工具执行与错误语义：
  - 通过进程内 `httpx.AsyncClient(ASGITransport)` 调用路由，超时 10 秒（`fastapi_mcp/server.py:115-119`）；
  - HTTP 4xx/5xx 被转为 `raise Exception("Error calling {tool}. Status code: …")`（`fastapi_mcp/server.py:558-561`）；
  - MCP SDK 1.26.0 把异常包装为 `CallToolResult(isError=True)`（`mcp-1.26.0-py3-none-any.whl:mcp/server/lowlevel/server.py:583-584,467-472`）；
  - 默认按 inputSchema 校验参数（同文件 `:492`），校验失败返回 `"Input validation error: …"`（同文件 `:532`）。
- 生成的环境代码被禁止写任何错误处理（`try/except`、`HTTPException` 等，`awm/prompts.py:234`）。推断：运行期错误大多以 HTTP 500 → `isError` 的形式出现（由上面两条源码事实推出，Phase 3 实测确认）。
- 所有用户相关操作隐式使用 `user_id=1`（`awm/prompts.py:62-63,227`；agent prompt `awm/core/agent.py:105`）。
- `env start` 的产物目录：`outputs/servers/<YYYYmmdd_HHMMSS>_<scenario>/`（`awm/core/server.py:45-51`），内含
  - `initial.db`（`:67-68` 或 `:85-87`）
  - `final.db`（`:79-81`，退出时再拷贝一次，`:166-169`）
  - `server_code.py`（`:152`）
  - `server.log`（`:161-163`）
- 数据库来源：未指定 `--db_path` 时，由 `gen_db.jsonl` + `gen_sample.jsonl` 现场建库（`awm/core/server.py:71-77` → `awm/core/reset.py:53-100` `reset_single_database`）。
- **副作用**：未指定 `--temp_server_path` 时，会把 `temp_server_<scenario>.py` 写到 `envs_load_path` 所在目录（`awm/core/server.py:134-138`）。若 `envs_load_path` 指向官方数据目录，就会往其中写文件。后续封装必须显式传 `--temp_server_path`。
- 进程模型：
  - `run_server` 用 `os.system(f"{python} {code} 2>&1 | tee {log}")` 阻塞运行（`awm/core/server.py:163`）。
  - 辅助函数 `start_server_process` 以 `Popen(stdout=PIPE, stderr=PIPE)` 启动 `python -m awm.core.server`（`awm/core/server.py:174-195`），`awm agent` 使用它（`awm/core/agent.py:445`），结束时只对该父进程 `terminate()`（`awm/core/agent.py:605-612`）。
  - 风险（推断，待 Phase 2 实测）：管道无人读取可能写满阻塞；真正的服务进程在子 shell 管道里，只终止父进程可能留下孤儿进程。
  - 对照：`check_all` 的测试路径使用 `start_new_session=True` + `os.killpg`（`awm/core/env.py:161-174,191,215`）。
- 健康检查（`awm env check`）：`check_mcp_server`（`awm/tools.py:141-199`）
  - 用 `mcp_agent` 的 `MCPApp` 以 streamable_http 连接，带超时执行 `list_tools`；
  - 至少有 1 个工具才判定为运行中（`awm/tools.py:193-195`）；
  - `wait_for_server` 每 0.5 秒轮询一次（`awm/tools.py:201-225`）。
- 端口工具：`get_random_available_port`（`awm/tools.py:96-116`）。`kill_process_on_port` 使用 `lsof` + `kill -9`（`awm/tools.py:227-235`），后续不复用。

### 1.3 `awm agent` 的 agent loop 与 prompt

- Config（`awm/core/agent.py:49-80`）：
  - `max_iterations=30`（`:64`）、`temperature=1.0`、`max_tokens=2048`；
  - `envs_path`、`tasks_path`、`db_path`、`sample_path` 均默认在 `./outputs/` 下（`:72-78`）。
- 两种运行方式（`awm/core/agent.py:395-458`）：
  - `--task` + `--mcp_url`：连接已有服务；
  - `--scenario` + `--task_id`：在输出目录建库，随机端口自动起服，最多等待 60 秒（`:433-454`）。
- System prompt（`awm/core/agent.py:88-127`）：
  - 只暴露两个元函数：`list_tools`、`call_tool`（`call_tool` 的参数为 `tool_name` + JSON 字符串 `arguments`）；
  - 工具调用格式为 `<tool_call>{"name": …, "arguments": …}</tool_call>`；
  - 要求先调用且只调用一次 `list_tools`，每步只调用一个函数，最后一步直接输出答案。
- 解析：
  - 用正则 `<tool_call>\s*(.*?)\s*</tool_call>` 从**文本**中解析（`awm/core/agent.py:130-167`，正则在 `:132`），兼容 `mcp_tool_` 前缀（`:154-159,179-180`）；
  - 工具列表渲染时加 `mcp_tool_` 前缀（`:243`）。
- LLM 调用：OpenAI 兼容的 `chat.completions`（`awm/core/agent.py:339-386`）。
  - 工具结果在 vLLM 模式下以 `role=tool` 回传，否则以 user 消息 `Tool response:` 回传（`:355-365`）。
  - vLLM 额外参数：`add_generation_prompt`、`min_tokens=16`、`chat_template_kwargs.enable_thinking=True`（`:374-379`）。
  - 是否启用 vLLM 额外参数用字符串启发式判断：`api_url` 同时包含 `"localhost"` 和 `"v1"` 才启用（`:417`）。写成 `127.0.0.1` 不会启用。
  - 推断：工具调用从文本解析，不依赖 OpenAI 原生 `tool_calls`，因此跑 `awm agent` 时 vLLM 不需要 `--tool-call-parser`。Phase 4 需对照模型卡确认。
- 循环（`awm/core/agent.py:490-571`）：
  - 某一步没有工具调用即结束（`:506-518`）；
  - 只执行第一个工具调用，其余记警告（`:520-528`）；
  - 达到上限记警告（`:570-571`）。
- 工具执行器 `MCPToolExecutor`：
  - 每次调用都新开一次 MCP app/session（`awm/core/agent.py:299-336`），超时 60 秒（`:273`）；
  - `isError` 结果以 `"Error: …"` 字符串返回（`:334-335`）。

### 1.4 `awm verify` 的两种模式

- Config（`awm/core/verify.py:36-49`）：`input`（run 目录）、`init_db_path`、`final_db_path`、`mode`（默认 `sql`）、`verifier_path`、`verifier_code_path`。
- 默认 verifier 文件（`awm/core/verify.py:376-379`）：
  - `code` 模式：`./outputs/gen_verifier.pure_code.jsonl`
  - `sql` 模式：`./outputs/gen_verifier.jsonl`
- `sql` 模式（推荐，code-augmented LLM-as-a-Judge）：
  1. 要求 LLM 环境变量已设置（`:52-78`）；
  2. `exec` 执行 verifier 代码，产出信息字典（`:104-148`）；
  3. 交给 LLM 裁判（`:230-334`），分类为 `complete / incomplete / server_error / agent_error`（`:250-260`），解析失败归为 `judge_error`（`:325-328`）。
- `code` 模式：`exec` 执行 verifier，返回 `{"result": "complete"|"others"}`（`:151-198`）。
- 函数名：扫描 `def verify_` 开头的定义，否则默认 `verify_task_completion`（code）或 `verify_task`（sql）（`:209-215`）。
- 安全性：
  - 两种模式执行期间都会把两个 DB 设为只读（`:111-117`，`:158-164`）；
  - 但以完整 `__builtins__` 调用 `exec`（`:120-126`，`:167-174`），即会执行数据集里的任意代码；
  - OpenEnv 对此有安全警告（[README.md#L243](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/README.md#L243)）。
- 输出：`<run_dir>/verify.<mode>.json`（`:436-437`），字段为 `scenario, task_id, task, mode, reward_type, verify_result`（`:407-414`），sql 模式另有 `llm_judge`（`:429-432`）。

### 1.5 `awm agent` 输出目录结构

- 路径：`outputs/agents/<YYYYmmdd_HHMMSS>`；scenario 模式下为 `outputs/agents/<ts>_<scenario>_task_<task_id>`（`awm/core/agent.py:419-427`）；可用 `--output_dir` 覆盖。
- 文件：
  - `trajectory.json`（`:591-592`）
    - 顶层字段：`scenario, task_id, task, model, api_url, max_iterations, temperature, total_iterations, timestamp, trajectory, messages`（`:578-590`）；
    - `trajectory[]` 条目：`iteration, role, content, tool_calls[], tool_response{tool_call_id, content}`，最后一条带 `is_final: true`（`:511-517`，`:560-569`）；
    - `messages[]` 的角色有 `system / user / assistant / tool`（`:482-485,503,553-557`）。
  - `initial.db` / `final.db`：仅 scenario 模式生成（`:441` 调用 `server._prepare_database`；结束时拷贝 `final.db`，`:595-599`）。
  - `server_code.py` / `server.log`：自动起服时由服务子进程写入同一目录（`awm/core/server.py:152,161`）。
  - `verify.<mode>.json`：运行 `awm verify` 后生成（`awm/core/verify.py:436`）。
- 注意：`--mcp_url` 模式不保存 DB（`db_file_path` 为 None），此时 `awm verify` 需要显式传 `--init_db_path/--final_db_path`（`awm/core/verify.py:41-43,367-368`）。

### 1.6 数据集的文件与字段

HF 数据集卡本身未能访问（见 §9）。以下字段来自**写入这些文件的源码**，文件名另以 OpenEnv 的加载器作旁证。

| 文件 | 写入位置 | 每行字段 |
|---|---|---|
| `gen_scenario.jsonl` | `awm/core/scenario.py:629` | `name, description`（`:465`），分类后另有 `suitability_level` 等字段（`:112`）；仓库自带种子 `outputs/seed_scenario.jsonl` 有 100 行 `{name, description}` |
| `gen_tasks.jsonl` | `awm/core/task.py:142` | `scenario, tasks[]`（`:94-97`），每场景 `num_tasks` 条（默认 10） |
| `gen_db.jsonl` | `awm/core/db.py:259` | `scenario, db_schema{tables[{name, ddl, indexes[], examples[]}]}, db_path`（`:194-198`；表结构键 `:85-94`） |
| `gen_sample.jsonl` | `awm/core/sample.py` | `scenario, tables_count, inserts_count, sample_data{tables[{insert_statements[]}]}`（`:224-227`；`:90-92`） |
| `gen_spec.jsonl` | `awm/core/spec.py:120` | `scenario, api_spec{api_groups[{endpoints[]}]}`（`:89-100`） |
| `gen_envs.jsonl` | `awm/core/env.py:568` | `scenario, db_path, full_code`（`:454-458`） |
| `gen_verifier.jsonl` / `gen_verifier.pure_code.jsonl` | `awm/core/verifier.py:175` | `scenario, task_idx, task, verification{code, raw_response}`（`:232-240`） |
| `databases/<scenario>.db` | `awm/core/pipeline.py:49`；`awm/core/db.py:66-111` | SQLite 文件 |

- 旁证：OpenEnv 从 HF 下载的 7 个文件（[data_loader.py#L20-L31](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/data_loader.py#L20-L31)）为 `gen_scenario / gen_tasks / gen_db / gen_sample / gen_envs / gen_verifier / gen_verifier.pure_code`，其中不含 `gen_spec.jsonl` 和 `databases/`。
- 下载命令（AWM README）：`hf download Snowflake/AgentWorldModel-1K --repo-type dataset --local-dir ./outputs/`（`README.md:60`）。
- `check_all` 会读取每条记录的 `db_path` 并复制该 DB（`awm/core/env.py:158`）。因此在未下载 `databases/` 的情况下，要先执行 `env reset_db`。
- 真实工具名旁证：OpenEnv 示例夹具捕获了 `e_commerce_33` 场景 task 0 的一次 rollout，其中列出 39 个工具（如 `search_products`、`add_item_to_cart`、`delete_user_payment_method`），见 [awm_ecommerce_episode.json](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/examples/echo_on_agent_world_model/fixtures/awm_ecommerce_episode.json)。这是第三方捕获的数据，**未对照数据集本身核实**。

### 1.7 Python 版本与依赖

- `requires-python = ">=3.12"`（`pyproject.toml:6`），`.python-version` 为 `3.12`（`.python-version:1`）。
- 运行依赖（`pyproject.toml:7-20`），其中硬性 `==` 固定：`fastapi==0.115.12`、`fastapi-mcp==0.4.0`、`mcp-agent==0.2.6`、`sqlalchemy==2.0.41`。
- `uv.lock`（lock version 1，247 个包）中的关键锁定版本：`mcp 1.26.0`、`pydantic 2.12.5`、`pydantic-settings 2.12.0`、`starlette 0.46.2`、`uvicorn 0.40.0`、`httpx 0.28.1`、`openai 2.38.0`、`numpy 2.4.2`、`typer 0.21.1`、`sse-starlette 3.0.3`。提取命令：用 `tomllib` 读取 `uv.lock` 的 `package` 表。
- `bench` extra 指向路径源 `mcp-adapted-bench`（`pyproject.toml:22-30`），该子模块未初始化。在它缺失的情况下 `uv sync` 能否成功，尚未验证（见 §9）。

---

## 2. AgentFly（Agent-One-Lab/AgentFly @ 1256586）

以下路径均相对 `third_party/AgentFly/`。

### 2.1 入口点与 CLI

- 没有 console script。入口为 `python -m agentfly.cli <train|deploy|swebench|search>`（`src/agentfly/cli.py:7-45`）。
- `train` 把参数原样交给 Hydra 脚本 `agentfly.verl.trainer.main_ppo`（`src/agentfly/cli.py:23-30`）。
  - `src/agentfly/verl` 是指向 `../../verl/verl` 的软链接（git mode 120000，`git ls-tree HEAD src/agentfly/verl`），即嵌套 submodule `verl`（`.gitmodules:1-3`）。
  - 该 submodule 的 URL 是 **SSH 形式** `git@github.com:Agent-One-Lab/verl.git`（`.gitmodules:3`），没有 SSH key 的环境需要配置 `insteadOf` 重写。
- 训练启动示例：`examples/train_scripts/train_example.sh:57-103`（GRPO，`:41`；Ray 集群，`:17-21`）。这是 GSM8K 数学任务示例，与 AWM 无关。

### 2.2 MCP / AWM 相关实现

- AgentFly main 以及另外 9 个分支的 tip 中都**没有** AWM 或 MCP 集成代码，检索结果见 §4。
- 分支 `resource` / `context` 只在设计文档中提到 MCP（例如 `docs/design/resource_engine_proposal.md`）。

### 2.3 Python 版本与依赖

- `requires-python = ">=3.12,<3.13"`（`pyproject.toml:30`），`license = { text = "Apache-2.0" }`（`pyproject.toml:31`）。
- 依赖（`pyproject.toml:33-56`）含 `vllm==0.19.0`（`:44`）；`verl` extra（`:58-80`）含 `flash-attn`（`:64`），其构建依赖 torch（`:98-99`）。
- **上游内部不一致**：`src/agentfly/requirements.txt:9` 固定的是 `vllm==0.10.0`，与 pyproject 不同。
- 安装流程（`install.sh:234-268`）：`git submodule init/update` → `pip install -e .` → `pip install -e '.[verl]' --no-build-isolation`。veRL **不会被 pip 安装**（通过软链接使用），所以 veRL `setup.py` 中的约束不会被强制执行。
- 嵌套 veRL fork @ 001f000：
  - 版本 `0.8.0.dev`（[verl/version/version](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/verl/version/version)）；
  - `install_requires` 含 `numpy<2.0.0`、`ray[default]>=2.41.0`、`tensordict>=0.8.0,<=0.10.0,!=0.9.0`（[setup.py#L26-L45](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/setup.py#L26-L45)）；
  - vLLM extra 为 `vllm>=0.8.5,<=0.12.0`（[setup.py#L52](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/setup.py#L52)）；
  - 另外，AgentFly README 写的是"verl 0.6.x"（`README.md:48`），与 fork 实际版本号不符。

---

## 3. OpenEnv `envs/agent_world_model_env`（@ e401886，仅侦察）

- 定位：把 AWM 环境包装成 OpenEnv 服务。每个 WebSocket 会话对应一个环境子进程，数据从 HF 下载（[data_loader.py#L16-L69](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/data_loader.py#L16-L69)）。
- 特殊工具 `verify` / `done` / `__list_scenarios__`（[README.md#L81-L84](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/README.md#L81-L84)）。
- 步级奖励映射（这是环境适配的一部分，不是训练配方）：
  - 默认值 `complete / incomplete / format_error`（[config.py#L31-L36](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/config.py#L31-L36)）；
  - `FORMAT_ERROR_TYPES`（[awm_environment.py#L37-L38](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/awm_environment.py#L37-L38)）；
  - `_get_reward`（[awm_environment.py#L464-L470](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/awm_environment.py#L464-L470)）。
- 进程管理：`start_new_session=True`（[scenario_manager.py#L234-L238](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/scenario_manager.py#L234-L238)）加 `killpg`（[#L367-L390](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/scenario_manager.py#L367-L390)）；代码补丁逻辑与 AWM 相同（[#L57-L80](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/server/scenario_manager.py#L57-L80)）。可作为 Phase 2 的参考实现。
- 依赖：`requires-python >=3.10`；`fastapi-mcp==0.4.0`、`mcp-agent==0.2.6`、`openenv>=0.2.2`、`gradio>=6.15.0`（[pyproject.toml#L9-L23](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/pyproject.toml#L9-L23)）。
- 社区示例 `examples/awm_expert_in_the_loop`、`examples/echo_on_agent_world_model` 都建立在该环境之上，但不是论文配方。例如 expert 示例使用 Qwen/Qwen3.5-4B（[README.md#L78](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/examples/awm_expert_in_the_loop/README.md#L78)）。

### 3.4 是否需要把 OpenEnv 纳入 submodule

结论：**当前不需要**。理由：

- 训练配方不在其中（§4）；
- 环境层直接复用 AWM 源码（§1.2）；
- 引入它会额外带来 `openenv`、`gradio` 等依赖。

本文件中的引用都使用固定 SHA 的永久链接。若 Phase 7 决定用它作为训练时的环境后端，届时再以 ADR 形式纳入。

---

## 4. 训练配方定位

**结论：(b) 只找到环境适配，没有完整配方。**（未根据论文文字自行复刻任何配方。）

| 位置 | 检索范围 | 方法 | 发现 |
|---|---|---|---|
| AgentFly | `main`@1256586 以及 `Simuphy, agents, algorithm, bilal/last_turn_mask_only, chatbricks, context, release, resource, shu-agentfly` 共 10 个分支的 tip | 浅拉取后 `git grep -i "awm\|agent_world\|agentworld\|world_model\|arctic\|snowflake\|mcp"` | 无 AWM 相关代码或配置；只有 `context`、`resource` 分支的设计文档与 lock 文件里出现 MCP 字样 |
| veRL fork | `main, agent, release, shu-agentfly` 分支，以及 AgentFly 固定的 `001f000` | 同上（不含 mcp） | 唯一命中是 README 采用者名单中的 "Snowflake"（[README.md#L211](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/README.md#L211)），与配方无关 |
| OpenEnv | `envs/agent_world_model_env` 与全仓 `examples/` | 通读 README 与 server 代码，全仓 grep | 只有环境适配和步级奖励映射（§3），没有 trainer 配置、超参数或启动脚本 |
| AWM | 全仓（4 个提交）以及嵌套的 `mcp-adapted-bench`@2dff8bd（只读） | 通读，grep `grpo/verl/ppo/kl_coef` 等 | AWM 本体没有训练代码。评测 harness 中的 `common/rule_reward.py` 实现了 `<think>` 格式规则（唯一词数上下限，以及 `hallucination_tool` 判为 `format_error`），由环境变量 `VERL_TRAIN_RULE_REWARD_THINK` 控制（[rule_reward.py#L4-L8](https://github.com/Raibows/mcp-adapted-bench/blob/2dff8bdfc35082e5b6980c4954b6074159a15854/mcp_adapted_bench/common/rule_reward.py#L4-L8)，[#L11-L30](https://github.com/Raibows/mcp-adapted-bench/blob/2dff8bdfc35082e5b6980c4954b6074159a15854/mcp_adapted_bench/common/rule_reward.py#L11-L30)，[#L55-L60](https://github.com/Raibows/mcp-adapted-bench/blob/2dff8bdfc35082e5b6980c4954b6074159a15854/mcp_adapted_bench/common/rule_reward.py#L55-L60)），在 τ² 评测中用于提前终止（[tau2/runner.py#L49](https://github.com/Raibows/mcp-adapted-bench/blob/2dff8bdfc35082e5b6980c4954b6074159a15854/mcp_adapted_bench/tau2/runner.py#L49)）。这只是奖励规则的一个片段，不是配方。变量名暗示它出自训练代码，但这只是推断 |

各处均**未找到**：AWM 训练用的 GRPO 超参数、滑动窗口历史的实现或配置、训练子集清单（论文所述的环境与任务划分）、启动脚本。

影响：按任务书 Phase 7，结论为 (b) 时**不创建** `paper_mirror` profile，并在文档中说明原因。

---

## 5. 论文数字核实（任务书 Phase 0 第 4 项）

- arXiv 2602.10090 的 PDF **无法打开**（`arxiv.org` 被出口策略拒绝，见 §0）。任务书 §1.4 中已有的数值已原样录入 `results/registry.yaml` 草稿，每条都是 `verified: false`，表号、行列与论文版本号留空（`null`），并注明来源为任务书。
- 旁证（非一手来源，不作为核实依据）：服务端 WebSearch 的摘要显示该论文存在 v2 版本，且摘要中出现的 3 个数值与任务书 §1.4 一致（对应 registry 中 `bfclv3.overall.8b.*`、`tau2.pass1.14b.awm`、`mcp_universe.success.8b.*`）。
- 论文的其他格子（τ²-bench 分领域、MCP-Universe 分类别、其余尺寸的基座值）均未能读取，列在 registry 的 `pending` 中。

## 6. 许可证调查（R6）

完整表格见 `docs/UPSTREAM.md` §3。摘要：

- **AWM 仓库：缺失许可证**。4 个提交的完整历史中从未出现 LICENSE / COPYING / NOTICE 文件（`git log --all --diff-filter=AD -- 'LICENSE*' …` 结果为空）；`pyproject.toml` 没有 license 字段；README 没有许可证声明。
  - 远端有一个未合并的 PR #17（第三方贡献者，2026-09-16），提议加入 MIT LICENSE。非维护者的提议不构成授权。
  - → **触发 R6 停止条件，需要仓库主人决定。**
- AgentWorldModel-1K 数据集卡、Arctic-AWM-4B/8B/14B 模型卡：**未能访问**（`huggingface.co` 被拦截）。
  - 旁证：OpenEnv 示例的 `SOURCE.md` 声称该数据集为 CC-BY-4.0（[SOURCE.md#L9](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/examples/echo_on_agent_world_model/fixtures/SOURCE.md#L9)）。这是第三方陈述，未核实。
- AgentFly：Apache-2.0（`LICENSE`；`pyproject.toml:31`）。
- veRL fork：Apache-2.0（LICENSE；[setup.py#L85](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/setup.py#L85)；Notice.txt 版权归 Bytedance）。
- OpenEnv：BSD-3-Clause（[LICENSE#L1-L3](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/LICENSE#L1-L3)，版权方 Hugging Face, Inc.；[pyproject.toml#L10](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/pyproject.toml#L10)）。
- mcp-adapted-bench（不使用）：根目录没有 LICENSE，pyproject 也没有 license 字段。

---

## 7. 依赖冲突分析（R11）

### 7.1 版本要求

| 组件 | Python | 关键约束 | 来源 |
|---|---|---|---|
| AWM | `>=3.12` | `fastapi==0.115.12`、`fastapi-mcp==0.4.0`、`mcp-agent==0.2.6`、`sqlalchemy==2.0.41`、`numpy>=2.4.2`、`openai>=2.17.0`；lock：`mcp 1.26.0`、`starlette 0.46.2`、`pydantic 2.12.5` | `third_party/agent-world-model/pyproject.toml:6-20`；`uv.lock` |
| fastapi-mcp 0.4.0 | `>=3.10` | `mcp>=1.12.0`、`fastapi>=0.100.0`、`pydantic>=2.0.0` | PyPI JSON `fastapi-mcp/0.4.0` |
| mcp-agent 0.2.6 | `>=3.10` | `mcp>=1.20.0`、`numpy>=2.1.3`、`pydantic>=2.10.4`、`fastapi>=0.115.6` | PyPI JSON `mcp-agent/0.2.6` |
| MCP Python SDK | `>=3.10` | 1.26.0：`pydantic>=2.11,<3`、`starlette>=0.27`；最新 2.2.0 改为依赖 `mcp-types`、`httpx2` | PyPI JSON `mcp/1.26.0`、`mcp`（latest） |
| AgentFly | `>=3.12,<3.13` | `vllm==0.19.0`（pyproject）；`vllm==0.10.0`（requirements.txt）；`[verl]` 含 `flash-attn`、`peft`、`ray[default]` | `third_party/AgentFly/pyproject.toml:30,44,58-80`；`src/agentfly/requirements.txt:9` |
| veRL fork 0.8.0.dev | — | `numpy<2.0.0`、`tensordict>=0.8.0,<=0.10.0,!=0.9.0`、`ray[default]>=2.41.0`；vllm extra `>=0.8.5,<=0.12.0` | [setup.py#L26-L52](https://github.com/Agent-One-Lab/verl/blob/001f000ae2e4cf05bb94c01427898cbe68961141/setup.py#L26-L52) |
| vLLM 0.19.0 | `>=3.10,<3.14` | `torch==2.10.0`、`transformers>=4.56.0,<5`、`pydantic>=2.12.0`、`openai>=2.0.0`、`fastapi[standard]>=0.115.0`、`mcp`（不限版本） | PyPI JSON `vllm/0.19.0` |
| LangGraph 1.2.12 | `>=3.10` | `langchain-core>=1.4.7,<2`、`langgraph-checkpoint>=4.1.0,<5`、`pydantic>=2.7.4` | PyPI JSON `langgraph` |
| langgraph-checkpoint-sqlite 3.1.1 | `>=3.10` | `aiosqlite>=0.20`、`sqlite-vec>=0.1.6` | PyPI JSON |
| langchain-mcp-adapters 0.3.2 | `>=3.10` | **`mcp>=1.24.0,<2.0.0`**、`langchain-core>=1.3.3,<2` | PyPI JSON |
| sse-starlette | — | 最新 3.4.11 要求 `starlette>=0.49.1`；AWM lock 使用 3.0.3 | PyPI JSON；AWM `uv.lock` |

### 7.2 冲突点

1. **numpy**：AWM 要求 `>=2.4.2`，veRL `setup.py` 要求 `<2.0.0`。若按 veRL 的约束装进同一环境则无解，这是必须拆成两个环境的直接原因。
2. **fastapi / starlette**：AWM 固定 `fastapi==0.115.12`，由此把 starlette 锁在 `<0.47`（lock 为 0.46.2）。app 环境的 API 层必须使用与之兼容的 SSE 实现：sse-starlette 需 ≤3.0.x（最新版要求 starlette ≥0.49.1），或改用 `StreamingResponse`。
3. **MCP SDK**：`langchain-mcp-adapters` 要求 `mcp<2.0.0`，AWM 锁 1.26.0，两者兼容。train 环境因 vLLM 对 `mcp` 不设上限而解析到 2.2.0，2.x 客户端与 1.26 服务端的互通性未验证。
4. **AgentFly 栈内部**：vLLM 三处约束互相矛盾（0.19.0 / 0.10.0 / ≤0.12.0）；实际解析得到的 `tensordict==0.14.2`、`numpy==2.2.6` 都超出 veRL `setup.py` 的约束。这是上游自身的不一致，Phase 1/7 不自行"修正"上游，只做记录。
5. **torch / vLLM**：只能进入 train 环境（以及独立的 serving 环境），app 环境不引入。

### 7.3 只解析不安装的求解验证（scratchpad，未修改仓库）

- app 候选：`agent-world-model @ file://…/third_party/agent-world-model` + `langgraph, langgraph-checkpoint-sqlite, langchain-mcp-adapters, pydantic-settings, typer, prometheus-client, httpx, pyyaml, pytest, pytest-asyncio, ruff, mypy`。
  - 命令：`uv pip compile app.in -c awm_lock_constraints.txt --python-version 3.12`，其中约束文件由 AWM `uv.lock` 的 242 个 registry 包版本生成。
  - 结果：**求解成功**。关键版本为 `mcp 1.26.0, fastapi 0.115.12, starlette 0.46.2, sse-starlette 3.0.3, pydantic 2.12.5, langgraph 1.2.12, langchain-core 1.6.4, langgraph-checkpoint-sqlite 3.1.1, langchain-mcp-adapters 0.3.2, openai 2.38.0`，即没有升级或降级 AWM 锁定的任何版本。
- train 候选：`agentfly[verl] @ file://…/third_party/AgentFly`。
  - 命令：`uv pip compile train.in --python-version 3.12 --python-platform x86_64-manylinux_2_28`。
  - 结果：**求解成功**。关键版本为 `vllm 0.19.0, torch 2.10.0, transformers 4.57.6, numpy 2.2.6, ray 2.58.0, tensordict 0.14.2, flash-attn 2.8.3.post1, peft 0.21.0, mcp 2.2.0`。
- 结论：拆成两个环境后，两边都不需要改动上游固定版本；app 环境可以完全落在 AWM 的锁定版本内。
- 注意：求解成功不等于安装成功。train 环境在 GPU 机器上的实际安装为 UNVERIFIED-LOCAL。
- Arctic-AWM 的 serving（vLLM）应独立于 app 环境。具体 vLLM 版本需以模型卡为准，模型卡目前无法访问。

---

## 8. 对后续阶段有直接影响的发现

| 阶段 | 发现 | 依据 |
|---|---|---|
| Phase 1 | app 环境需要 Python 3.12，本机有 `/usr/bin/python3.12`，uv 的 python-build-standalone 下载地址可达（HTTP 206） | `pyproject.toml:6`；`uv python list --show-urls` |
| Phase 1 | mock 夹具的工具名需取自真实数据集。数据集目前无法下载，只有 OpenEnv 捕获的 `e_commerce_33` 工具清单可作旁证 | §1.6 |
| Phase 2 | 服务进程本身只能以子进程运行，因为 `run_server` 会阻塞（`awm/core/server.py:163`）。建库可直接调用 Python 接口 `reset_single_database`（`awm/core/reset.py:53-100`），健康检查可复用 `check_mcp_server`（`awm/tools.py:141-199`）。启动时必须显式传 `--db_path`、`--temp_server_path`、`--output_dir` | §1.2 |
| Phase 2 | `check_all` 依赖 `db_path` 已存在（`awm/core/env.py:158`） | §1.6 |
| Phase 3 | 上游错误以 `isError` 文本返回，有两种形态：`Error calling … Status code …` 和 `Input validation error: …` | §1.2 |
| Phase 4 | `awm agent` 从文本解析 XML 形式的工具调用，并开启 `enable_thinking`；vLLM 额外参数的启用依赖 `"localhost"` 字符串 | §1.3 |
| Phase 6 | AWM 与 OpenEnv 的 `trajectory.json` 格式不同（`awm/core/agent.py:578-590` 对照 [OpenEnv README#L132](https://github.com/meta-pytorch/OpenEnv/blob/e401886d23aab1be92493ea15e5d7e2cdf7e657b/envs/agent_world_model_env/README.md#L132)） | §1.5 |
| Phase 7 | `gen scenario` 会把分类结果写回**输入的种子文件**（`awm/core/scenario.py:642`），必须先把种子复制到 `data/synth/<run_id>/` 再运行，否则会改动子模块内文件（R5） | — |
| Phase 7 | 各 gen 步骤的续跑能力不一致：`gen env` 能续跑（`awm/core/env.py:120-133,335-359`）；`gen scenario` 有 `--resume`（`awm/core/scenario.py:48`）；`gen verifier` 会加载已有结果（`awm/core/verifier.py:465`）。其余步骤需要由编排层自己做 checkpoint | — |
| Phase 7 | AgentFly 的嵌套 veRL submodule 用 SSH URL（`third_party/AgentFly/.gitmodules:3`） | §2.1 |

## 9. 未能确认清单

1. arXiv 2602.10090 PDF：全部数值、论文版本号、表号与行列。**原因**：`arxiv.org` 被出口策略拒绝。
2. HF 数据集卡 `Snowflake/AgentWorldModel-1K`：许可证、实际文件清单与大小、场景数与任务数。**原因**：`huggingface.co` 被拒绝。
3. HF 模型卡 `Snowflake/Arctic-AWM-4B/8B/14B`：许可证、基座模型、chat template、推荐的 vLLM 版本与 tool parser、上下文长度。**原因**：同上。
4. AWM 仓库的授权状态：没有 LICENSE，上游维护者也没有任何声明（见 §6）。
5. 运行期行为（Phase 1/2 实测）：
   - `awm agent` 起服后只终止父进程，是否会留下孤儿服务进程；
   - PIPE 管道写满后是否阻塞；
   - `mcp-adapted-bench` 缺失时 `awm --help`（`simpleArgParser` 遇到 `None` 配置）和 `uv sync` 是否正常。
6. MCP SDK 2.x 客户端与 AWM 服务端（1.26.0）的互通性。
7. train 环境在 GPU 上的实际安装与运行（UNVERIFIED-LOCAL）。
8. 官方训练配方是否以非公开形式存在：公开渠道未找到（§4）。
9. OpenEnv 示例对数据集许可证（CC-BY-4.0）的陈述、以及其夹具中的工具清单，都是第三方信息，未对照 HF 原文核实。

---

## 10. 后续阶段补充核实（Phase 1 起，按阶段追加）

### Phase 1

- 已安装版本（`uv run python -c "import importlib.metadata as m; ..."`）：`mcp 1.26.0`、`langgraph 1.2.12`、`langgraph-checkpoint-sqlite 3.1.1`、`langgraph-checkpoint 4.2.0`、`langchain-core 1.6.4`、`fastapi 0.115.12`、`fastapi-mcp 0.4.0`、`mcp-agent 0.2.6`、`pydantic 2.12.5`、`starlette 0.46.2`、`sse-starlette 3.0.3`、`openai 2.38.0`、`typer 0.21.1`。
- §9 第 5 条中的一部分已实测确认：`mcp-adapted-bench` 未初始化时，`uv sync` 与 `uv run awm --help` 都能正常运行。
- `simpleArgParser` 0.2.2 会在解析后调用 dataclass 的 `pre_process()`（`simpleArgParser/s_argparse.py:563-572`）。
- train 环境 `uv lock` 解析出 244 个包，与 Phase 0 的只解析求解一致：`vllm 0.19.0`、`torch 2.10.0`、`numpy 2.2.6`、`tensordict 0.14.2`。

### Phase 2

- MCP SDK 1.26.0 客户端 API：
  - `streamablehttp_client(url, headers, timeout, sse_read_timeout, ...)`（`mcp/client/streamable_http.py:686-693`）；
  - `ClientSession.initialize / call_tool / list_tools`（`mcp/client/session.py:148,368,505`）。
- 用手写迷你场景对 AWM 启动器做了实测（`python -m awm.core.server --db_path … --temp_server_path … --output_dir …`）：
  - `list_tools` 返回全部 7 个路由的 operationId；
  - 枚举参数非法时返回 `isError=True`，文本为 `Input validation error: 'bogus' is not one of ['price', 'rating']`；
  - 缺少必填参数时返回 `Input validation error: 'product_id' is a required property`；
  - 服务端抛异常时返回 `Error calling get_product_by_id. Status code: 500. Response: Internal Server Error`；
  - 查询无结果时返回 `isError=False`，文本为 `'[]'`。
- §9 第 5 条的孤儿进程风险已实测证实：只终止 `python -m awm.core.server` 这个 launcher，`sh -c … | tee` 以及真正的服务进程都会继续存活。按进程组 `killpg` 之后，三者都能被杀死。本沙箱的 PID 1（`/process_api`）不回收僵尸进程，因此被杀的进程会以 `Z` 状态残留，测试只断言没有存活的非僵尸进程（`workbench/envs/procs.py`）。

### Phase 3

- MCP SDK 1.26.0 服务端：
  - lowlevel `Server.call_tool(*, validate_input=True)`（`mcp/server/lowlevel/server.py:492`）；
  - 处理器可以直接返回 `CallToolResult`（`:539-540`）；
  - 请求上下文里的 `request` 就是 HTTP 请求对象（`:758`）；
  - `StreamableHTTPSessionManager(app, event_store, json_response=False, stateless=False, ...)`（`mcp/server/streamable_http_manager.py:60-65`）。
- Starlette 的 `Mount("/mcp")` 访问 `/mcp` 时会 307 重定向，MCP 客户端不跟随，所以网关用 `Route` 挂一个 ASGI 类实例。
- 网关集成测试链路：迷你场景（AWM 启动器）→ 网关 MCP 前端 → MCP 客户端。实测结果：
  - 带 `X-Workbench-Session` 的 `list_tools` 返回带前缀的工具名；
  - destructive 工具在没有令牌时被拒绝，并写入审计；
  - 带令牌调用成功，DB diff 中 `payment_methods` 被删除的主键为 2。
