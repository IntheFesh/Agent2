# 2026-09-24 Phase 9 前置检查

- 机器：Claude Code 云端容器（Linux x86_64，无 GPU），出口经代理。
- 访问时间：2026-09-24 05:37–05:42 UTC。
- 原始输出：`docs/verification/logs/2026-09-24-preflight-network.log`。

## 1. 网络

| 目标 | 结果 | 说明 |
|---|---|---|
| `https://huggingface.co/api/datasets/Snowflake/AgentWorldModel-1K` | HTTP 200 | 第一次检查（05:37）被代理拒绝（CONNECT 403）；仓库主人调整网络策略后重试通过 |
| `https://arxiv.org/abs/2602.10090v3` | HTTP 200 | 同上。注意：WebFetch 工具仍被拦截，只能用 curl 访问 |
| `https://github.com` | 可达 | GitHub 工具与 git push 正常 |
| `hf download`（匿名，无 `HF_TOKEN`） | 成功 | 下载 LFS 文件 `gen_scenario.jsonl`：1338786 字节，1000 行 |

结论：满足 TASK_v2 Phase 9 第 1 步的要求。

## 2. CI

- 本容器没有 `gh` CLI，改用 GitHub API（Actions `list_workflow_runs`）查询。
- `f140123` 触发的是 run 10（`https://github.com/IntheFesh/Agent2/actions/runs/35960123474`）：`status: completed`，`conclusion: success`。
- 结论：CI 通过，无需修复。

## 3. 分支（N4）

- 从 `f140123` 新建 `phase9-verification`：`git checkout -b phase9-verification`。
- 本机 git 配置：`user.name=Claude`，`user.email=noreply@anthropic.com`（未修改）。
- 远端此前只有 `claude/kind-gauss-3clgyp` 一个分支。

## 4. N5 授权

三项都未勾选，因此：

- 不创建 `main`，不修改默认分支；
- 本轮结束时不开 PR，只推送 `phase9-verification`；
- 不在 AWM 仓库发 issue，只写草稿 `docs/verification/awm-license-issue-draft.md`（Phase 10）。

## 5. 各阶段的环境条件

| 阶段 | 需要 | 本容器现状 | 预判 |
|---|---|---|---|
| 10 外部事实核对 | arXiv、HF 页面 | 可访问（curl） | 可执行 |
| 11 官方数据集 | HF 下载，8 个数据文件合计 518753502 字节（HF API `tree` 接口列出的大小之和） | 可下载 | 可执行 |
| 12 真实 LLM 链路 | OpenAI 兼容端点与 key | `api.deepseek.com` 可达；**环境中没有 `DEEPSEEK_API_KEY` / `OPENAI_API_KEY`** | 需要仓库主人以环境密钥形式提供 key，否则阻塞 |
| 13 合成流水线 | LLM key；`gen scenario` 还需要 embedding 端点（`awm/core/scenario.py:63`） | 同上；DeepSeek 价格页没有 embedding 模型 | 无 key 时阻塞；有 key 也只能走"跳过场景生成"分支，需先确认 CLI 是否支持 |
| 14 Docker | Docker daemon，能拉取基础镜像 | daemon 原先未运行；以 root 手动启动 `dockerd` 成功（Server Version 29.3.1） | 预计可执行；基础镜像能否拉取到 Phase 14 再确认 |
| 15 GPU | CUDA GPU | 没有 `nvidia-smi` | 整阶段跳过 |

## 6. 本轮计划触碰的文件

| 阶段 | 文件 |
|---|---|
| 9 | `TASK_v2.md`（新增），`docs/verification/2026-09-24-preflight.md`，`docs/verification/logs/2026-09-24-preflight-network.log` |
| 10 | `results/registry.yaml`，`src/workbench/results/registry.py`（仅当新字段或表注需要渲染支持时），`docs/RESULTS.md`（生成），`docs/UPSTREAM.md`，`README.md`，`docs/LIMITATIONS.md`，`docs/verification/2026-09-24-paper-table4.md`，`docs/verification/2026-09-24-licenses.md`，`docs/verification/2026-09-24-model-cards.md`，`docs/verification/awm-license-issue-draft.md`，对应测试 |
| 11 | `scripts/download_data.sh`（仅注释），`tests/fixtures/awm_mini/**`，`tests/integration/test_official_data.py`（新增），`pyproject.toml`（注册 `official_data` marker），`tests/conftest.py`，`configs/number_whitelist.yaml`（如需要），`docs/verification/2026-09-24-dataset.md`，`docs/verification/e_commerce_33-tools.*`，`docs/LIMITATIONS.md`，`docs/RECON.md` |
| 12 | `docs/verification/logs/*`，`docs/verification/cost-ledger.md`，`docs/WALKTHROUGH.md`，`docs/LIMITATIONS.md`；仅在出现兼容性问题时修改 `src/workbench/llm/**` 并补单测和 ADR |
| 13 | `configs/pricing.yaml`，`docs/verification/logs/*`，`docs/verification/cost-ledger.md`，`docs/LIMITATIONS.md` |
| 14 | `docs/verification/logs/*`；仅在构建失败时修改 `Dockerfile` / `docker-compose.yml` 并补 ADR；`docs/LIMITATIONS.md` |
| 收尾 | `docs/LIMITATIONS.md`，`README.md`，`docs/CHANGELOG.md`，`CLAUDE.md` 状态段 |

## 7. API 费用预估（N3，上限 ¥30）

价格来源：DeepSeek 官方价格页 `https://api-docs.deepseek.com/zh-cn/quick_start/pricing/`，2026-09-24 用 curl 读取。原文摘录：

> 模型 deepseek-flash (1) deepseek-v4-pro … 百万tokens输入 （缓存未命中） 空闲时段 1元 4.5元 高峰时段 2元 9.0元 百万tokens输出 空闲时段 4元 13.5元 高峰时段 8元 27.0元
> (2) 空闲时段价格为高峰时段价格的一半。

估算按 `deepseek-flash` 的**高峰价**（输入 2 元 / 百万 tokens，输出 8 元 / 百万 tokens），不计缓存折扣，再乘以 3 倍安全系数：

| 调用 | token 量假设 | 估算 | ×3 上界 |
|---|---|---|---|
| Phase 12 `workbench agent run` ×1 | 约 15 次 LLM 调用，输入约 60K、输出约 5K | 约 ¥0.2 | ¥0.6 |
| Phase 12 `awm agent` ×1 | 最多约 20 轮，上下文逐轮增长，输入约 200K、输出约 10K | 约 ¥0.5 | ¥1.5 |
| Phase 12 `awm verify --mode sql` ×1 | 1–2 次调用，输入约 10K、输出约 2K | 不到 ¥0.1 | ¥0.2 |
| Phase 13 合成 1 个场景 | 6 个生成步骤，含代码生成与自动修复重试，输入约 300K、输出约 150K | 约 ¥1.8 | ¥5.4 |
| 合计 | | 约 ¥2.6 | 约 ¥7.7 |

预计不会超过上限。实际花费在 `docs/verification/cost-ledger.md` 中逐次记账；累计接近上限时停下询问。
