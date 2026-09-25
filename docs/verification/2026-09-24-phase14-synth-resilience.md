# 2026-09-24 合成运行的中断续跑与预算熔断（Phase 14，D17）

- **零成本**：没有调用任何付费 API，`DEEPSEEK_API_KEY` 从每个进程的环境中移除（D14）。
- **机器**：Claude Code 云端容器（Linux x86_64，无 GPU）。
- **设计**：ADR-022（中断与续跑）、ADR-023（预算熔断）。
- **日志**：`docs/verification/logs/2026-09-24-phase14-synth-resilience.log`。
- 本文只记录工程事实，不是评测。

## 1. 要解决的问题

- **中断**：Phase 13 的故意中断没有落在 gen 步骤中间。
  - 信号发给了包装 shell 的进程组，没有发给运行所在的进程组；
  - AWM 测试 server 在中断后成为孤儿，只能手动清理（`docs/verification/2026-09-24-synth.md`）。
- **预算**：账本只在运行结束后汇总，执行过程中没有上限。

## 2. 单测：真实子进程，假上游监听真实本地端口

`tests/unit/test_synth_resilience.py` 驱动 `tests/unit/synth_harness.py`：
- runner 作为独立进程运行，测试向它的 PID 发 SIGTERM；
- 每个步骤是真实子进程，经 workbench 代理访问假上游；
- 假步骤的行为与 AWM 一致：出错时写空回复并以 0 退出；`verifier` 边收边追加写入。

```
$ PYTHONPYCACHEPREFIX=$PWD/.cache/pycache uv run pytest -v tests/unit/test_synth_resilience.py
tests/unit/test_synth_resilience.py::test_interrupt_mid_step_then_resume[db-gen_db.jsonl] PASSED
tests/unit/test_synth_resilience.py::test_interrupt_mid_step_then_resume[verifier-gen_verifier.jsonl] PASSED
tests/unit/test_synth_resilience.py::test_budget_stop_fails_the_step_and_a_higher_budget_resumes PASSED
tests/unit/test_synth_resilience.py::test_proxy_refuses_over_budget_but_serves_the_cache PASSED
tests/unit/test_synth_resilience.py::test_proxy_fails_closed_without_a_price_and_can_be_disabled PASSED
tests/unit/test_synth_resilience.py::test_ledger_spent_ignores_cached_and_refused_entries PASSED
============================== 6 passed in 15.69s ==============================
```

**反向对照**：每次临时改动一处，跑完即恢复。

- 恢复旧的步骤启动方式（`subprocess.run`，与 runner 同一进程组）：两个中断用例都在"步骤启动的 server 已被回收"这一断言处失败。
- 去掉"重做前移开输出"：`verifier` 中断用例和预算用例失败。

**CI 上的一次失败及修复**：

- 现象：[ci run 27](https://github.com/IntheFesh/Agent2/actions/runs/36067444876)（提交 `c2a8a08`）中，`test_interrupt_mid_step_then_resume[verifier]` 以 `assert 3 < 3` 失败：runner 退出时，该步 3 个请求都已发出。
- 原因在测试的设计：
  - 测试在步骤收到第 1 个响应后发信号，把假上游 0.3 s 的延迟当作中断必须落进去的窗口；
  - 代理运行在 runner 进程里，停止时会等在途请求完成并记账（uvicorn 0.40.0 的停止过程，RECON "Phase 14"）；
  - 所以中断只要晚到约 250 ms，第 3 个请求就已发出并被记账。
- 本地复现：在测试驱动里把 runner 的 SIGTERM 处理推迟 0.5 s，两个用例都以同样的断言失败。
- 反应时间的量级：一个探针测得，另一个线程持续执行 Python 代码时，主线程阻塞在 `waitpid()` 中，从发信号到处理函数运行最长 0.274 s（40 次试验，全部在子进程结束前处理）。
- 修复（提交 `94a743e`）：
  - 假上游在被中断步骤的第 2 个请求第一次到达时把它保持 3 s，信号一定落在这个请求等待期间；
  - 推迟 0.5 s、1.5 s 后两个用例都通过，反向对照仍然失败；
  - [ci run 28](https://github.com/IntheFesh/Agent2/actions/runs/36068916710) 通过。
- runner 本身没有改动：从收到信号到停止步骤存在反应时间，其间步骤可能多走几步。续跑时这些结果由缓存重放或重新生成，不影响正确性。

## 3. 用真实 AWM 重做 Phase 13 的中断（本地回放上游）

**设置**：

- 回放上游：
  - 按请求字节原样查找 Phase 13 中 DeepSeek 给出的响应（`data/synth/p13_it_service_desk/llm_cache`），延迟 1.5 s 后返回；
  - 其它请求一律返回 HTTP 404，这样任何不确定性都会暴露出来，不会被掩盖。
- workbench：提交 `c2a8a08`，`src/` 之后没有改动。
- 运行：运行目录 `data/synth/p14_replay`（不入库）。与 Phase 13 相同的手写场景，同一条命令执行两次：

  ```
  workbench synth run --scenarios 1 --out data/synth/p14_replay --scenario-file data/synth/p14_replay/local_scenario.jsonl --execute
  ```

**第 1 次（被中断）**：

- 触发时机：`gen env` 的 LLM 响应已记账（1 行），AWM 正在测试它生成的 server（pid 4545）。此时向 runner 自己的 PID（4484）发 SIGTERM。
- 结果：
  - 退出码 130；`task`、`db`、`sample`、`spec` 已完成，`env` 记为 `interrupted`；
  - AWM 的测试 server 已被回收，没有进程残留；
  - AWM 的临时目录 `/tmp/env_test_1b52f5e8hrgxjqa7` 残留。这在 ADR-022 的代价中写明（AWM 被 SIGTERM 结束时不执行 `finally`），已手动删除。
- 中断前回放上游收到 5 个请求，全部是 Phase 13 记录过的。

**第 2 次（同一命令续跑）**：

- 退出码 0，6 个步骤都记为完成。
- 前 4 个步骤按 checkpoint 跳过；`env` 的请求由缓存命中（账本新增 1 行，`cached: true`），没有到达上游。
- 与 Phase 13 的产物对比：

  | 文件 | 结果 |
  |---|---|
  | `gen_scenario.jsonl`、`gen_tasks.jsonl`、`gen_sample.jsonl`、`gen_spec.jsonl` | 完全相同 |
  | `gen_db.jsonl`、`gen_envs.jsonl` | 只有 `db_path` 一个字段不同，它是运行目录的绝对路径 |
  | 收尾验证 `validation.json` | 完全相同：1 个环境启动成功，15 个工具 |
  | `gen_verifier.jsonl` | 10 行都没有代码（Phase 13 为 10 行都有代码），原因见下 |

- 这次运行的账本合计 ¥0.7306。这是按回放响应中原样带回的 `usage` 计价的结果；这些响应在 Phase 13 已经付过费，这次没有任何请求到达 DeepSeek。

**`gen verifier` 为什么全是空行**：

- verifier 的 prompt 嵌入了数据库内容（`awm/core/verifier.py:61-80`，调用处 `:183-189`）；
- 样例数据用相对时间生成：Phase 13 的 `gen_sample.jsonl` 中有 141 处 `datetime('now', …)`；
- 因此每次运行的数据库内容不同，请求正文与 Phase 13 不同，回放上游对 50 个请求全部返回 404。其中 AWM 对每个请求尝试 3 次，另有 25 次重复发送的来源没有查明；
- AWM 把失败的请求变成空回复（`awm/gpt.py:195-206`），写出 10 行没有代码的结果，然后以 0 退出；
- runner 只看"退出码为 0 且输出文件存在"，所以把 `verifier` 判为完成。

**由此暴露的限制**（已写入 LIMITATIONS §6）：

- runner 判定步骤成功的依据不够：上游持续出错时，AWM 仍以 0 退出并写出空结果，这个步骤会被记为完成。回放中的错误是 404；真实环境中可能是余额不足、限流或网络故障。
- 预算熔断的拒绝不受这个问题影响：runner 按账本中的 `refused` 条目判定失败（ADR-023）。
- 代理只记录上游成功的响应，所以 runner 目前看不到上游错误。是否要把"步骤内有请求最终失败"也判为步骤失败，列为 Phase 14 报告中的问题。
