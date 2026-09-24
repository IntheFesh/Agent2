# 2026-09-24 Docker 构建与 compose 启动（U6，Phase 14）

按仓库主人的决定 D15，先在当前容器里尝试启动 `dockerd`；拉不到镜像时，改在 GitHub Actions 的托管 runner 上验证。结论：**U6 已验证**。两次 CI 运行都完成了"构建 → 启动（不带 `gpu` profile）→ 经 HTTP API 走完 查询 → 写操作 → 审批 → 完成 → 停止"。

本文的数字都是工程事实（R3）：各为 1 次测量，在 GitHub 托管 runner 上测得，随 runner 与缓存状态变化，不代表任何部署环境的性能。日志摘录：`docs/verification/logs/2026-09-24-phase14-docker-smoke.log`。

## 1. 本地尝试（Claude Code 云端容器）

- `dockerd` 29.3.1 能启动（"Daemon has completed initialization"）。
- 构建时拉取基础镜像 `docker.io/library/python:3.12-slim` 失败，`registry-1.docker.io` 两次返回 `429 Too Many Requests`（日志 §1）。
- 即使能拉到镜像，沙箱说明（`/root/.ccr/README.md` 的 "docker build / docker run" 一节）也写明：容器里的进程连不到沙箱的出口代理，也不信任它的 CA，`uv sync` 下载依赖需要改 Dockerfile 安装沙箱 CA。这种改动只对这个沙箱有意义，不属于 D15 允许的"为让构建通过而修改"，因此没有做。
- 按 D15 改用 GitHub Actions，没有停下等待。

## 2. CI 任务

- 工作流：`.github/workflows/docker-smoke.yml`，push 到 `phase9-verification` 与 `main` 时运行，`ubuntu-latest`，超时 30 分钟，检出时带子模块（镜像要复制 `third_party/agent-world-model`）。
- 脚本：`python3 scripts/docker_smoke.py`，只用标准库。步骤：
  1. 准备：
     - 把手写迷你数据集 `tests/fixtures/awm_mini/gen_*.jsonl` 复制到 `./data/awm1k`；
     - 写一个临时 `.env`，选用脚本化的 mock LLM（`tests/fixtures/trajectories/demo_query_write_approve.jsonl`，每个会话从头回放）；
     - `./data` 或 `.env` 已存在时拒绝运行，不覆盖真实数据或用户配置。
  2. `docker compose build`，计时。
  3. 用 `docker compose config --images` 列出镜像，用 `docker image inspect --format {{.Id}} {{.Size}}` 取大小。
  4. `docker compose up -d`（不带 `gpu` profile，只起 `env-manager` 与 `app`），计时到 `GET /healthz` 返回 `ok`。
  5. 经 HTTP API 走一遍：
     - `POST /sessions`：启动 `mini_e_commerce`；
     - `POST /sessions/{id}/messages`：要求"把 200 美元以内最好的降噪耳机加入购物车"。智能体先调用读工具，写工具因需要审批而暂停（`approval_required`）；
     - `GET /approvals`：看到这条待审批；
     - `POST /approvals/{id}`：由 `ci-docker-smoke` 批准，写工具执行，会话结束（`done`）；
     - `GET /sessions/{id}/diff`：`cart_items` 表新增 1 行；
     - `DELETE /sessions/{id}`。
  6. `docker compose down`，删除临时的 `./data` 与 `.env`，打印汇总。
- Dockerfile 与 `docker-compose.yml` 没有为这项检查做任何修改，因此不需要新的 ADR。

## 3. 结果

| | run 1 | run 2 |
|---|---|---|
| 链接 | [actions/runs/36064190322](https://github.com/IntheFesh/Agent2/actions/runs/36064190322) | [actions/runs/36067444909](https://github.com/IntheFesh/Agent2/actions/runs/36067444909) |
| 提交 | `5419a2c` | `c2a8a08` |
| runner 镜像 | `ubuntu-24.04`，版本 `20260920.314.1` | 同左 |
| Docker / Compose | 28.0.4 / 2.38.2 | 同左 |
| 子模块 | AgentFly `1256586`，agent-world-model `85e322f` | 同左 |
| `docker compose build` | 15.9 s | 23.0 s |
| 镜像大小 | 880.1 MB | 880.2 MB |
| 冷启动（`up -d` 到 `/healthz` 为 ok） | 13.4 s | 15.0 s |
| `POST /sessions`（起 `mini_e_commerce`） | 5.4 s，7 个工具 | 7.6 s，7 个工具 |
| API 流程 | 通过 | 通过 |

run 1 的汇总（日志 §2，原样）：

```
== docker smoke summary ==
docker 28.0.4
compose 2.38.2
build (docker compose build): 15.9 s
image agent2-env-manager: 880.1 MB (sha256:a6e6825d8841)
image agent2-app: 880.1 MB (sha256:a6e6825d8841)
cold start (docker compose up -d -> GET /healthz ok): 13.4 s
session start (POST /sessions, mini_e_commerce): 5.4 s, 7 tools
query: mini_e_commerce__search_products -> ok
write paused for approval: mini_e_commerce__add_item_to_cart (risk write)
approved by ci-docker-smoke: mini_e_commerce__add_item_to_cart -> ok
done: "I added 'Wireless Noise Cancelling Headphones A' ($189) to your cart (quantity 1)."
db diff: cart_items added [2]
result: PASS
```

**测量口径**：

- 时间都来自脚本里的 `time.monotonic()`：
  - 构建：`docker compose build` 命令的墙钟时间；
  - 冷启动：从执行 `docker compose up -d` 到 `GET /healthz` 第一次返回 `{"ok": true}`。其中包括 compose 等待 `env-manager` 健康检查通过的时间，健康检查间隔 10 s；
  - 会话启动：`POST /sessions` 的耗时，含启动环境 server。
- 大小是 `docker image inspect` 的 `Size` 除以 10^6，即未压缩大小（MB，十进制）。

**观察**：

- `app` 与 `env-manager` 用同一个 Dockerfile（compose 中两者都是 `build: .`），两次运行各自只构建出一个镜像，挂两个名字，镜像 ID 相同。所以实际只有一个约 880 MB 的镜像。
- 两次运行的基础镜像 digest 相同：
  - `python:3.12-slim@sha256:2f17fc04…`；
  - `ghcr.io/astral-sh/uv:0.8.17@sha256:e4644cb5…`。
- `uv sync --frozen --no-dev` 安装了 121 个包。
- 两次运行都有一条非阻断警告：`actions/checkout@v4` 以 Node.js 20 为目标，被强制在 Node.js 24 上运行。

## 4. 没有验证的部分

- `gpu` profile 中的 `vllm` 服务：需要 NVIDIA GPU，仍是 U1，留到 Phase 15。
- 用官方数据集与真实 LLM 的容器化运行：本检查只用手写迷你数据集与 mock LLM。
- 其它平台：arm64、macOS 上的 Docker Desktop、Podman 都没有试过。
