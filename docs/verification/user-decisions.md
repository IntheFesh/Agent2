# 仓库主人的决定记录（TASK_v2 起）

按时间顺序记录仓库主人在对话中给出的决定。本文件只记录决定，不包含任何密钥。

## 1. Phase 9 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D1 | Phase 12、13 是否执行 | **执行**。仓库主人已在云环境中添加专用小额的 `DEEPSEEK_API_KEY`。 |
| D2 | 密钥使用方式 | `DEEPSEEK_API_KEY` 只在**进程环境**里映射给需要的变量（如 `OPENAI_API_KEY`、`OPENAI_BASE_URL`、本仓库 `openai_compat` 后端的配置）；**不得写入任何文件、日志或提交**。 |
| D3 | `awm agent` 与 DeepSeek 输出格式不兼容时 | 如实记录失败原因和日志即可；**不修改上游，也不临时写适配器**。本仓库 `src/workbench/llm/` 内的兼容修复仍按 TASK_v2 Phase 12 第 4 条执行（补单测与 ADR）。 |
| D4 | 费用上限 | 保持 **¥30**；累计花费超过 **¥15** 时，在阶段报告中提示一次。 |
| D5 | Phase 13 场景来源 | 没有 embedding 端点，**跳过场景生成**：先读源码确认 `awm gen task --input <scenario.jsonl>` 能直接以场景文件为输入；**手写 1 条企业类场景**（例如公司内部 IT 工单系统），名称加 `local_` 前缀，不复用官方场景名，格式与官方 `gen_scenario.jsonl` 一致，放在 `data/synth/<run_id>/` 下。CLI 不支持时停下报告。 |

## 2. N5 授权状态

| 项目 | 状态 | 执行情况 |
|---|---|---|
| 用 `claude/kind-gauss-3clgyp` 当前 HEAD 创建 `main` 并设为默认分支 | **已授权**（Phase 9 报告之后） | `main` 已于 2026-09-24 从 `f140123` 创建。默认分支的切换由仓库主人自己在 GitHub 设置中完成（Phase 10 报告之后的决定），Claude Code 不做。 |
| 本轮结束后从 `phase9-verification` 向 `main` 开 PR | **已授权** | 2026-09-25 本轮收尾时执行（D22）。 |
| 在 Snowflake-Labs/agent-world-model 开 issue 询问许可证 | **未授权** | 仓库主人用自己的账号发送（Phase 10 报告之后的决定）；Claude Code 不发送。草稿见 `docs/verification/awm-license-issue-draft.md`，其中的 "#17" 为纯文本（在上游仓库发出时会自动链接到其 PR #17），未改动。 |

## 3. Phase 10 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D6 | 默认分支 | 仓库主人自己在 GitHub 设置中改为 `main`。 |
| D7 | AWM 许可证 issue | 仓库主人用自己的账号发；N5 第 3 项保持未授权。 |
| D8 | Phase 12 的前置条件 | Phase 11 完成后如果仍看不到 `DEEPSEEK_API_KEY`，在报告中说明并停下；仓库主人会开新会话继续 Phase 12。 |

## 4. Phase 12 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D9 | 修复阶段 Phase 12.5 | 进入 Phase 13 之前先只修缺陷、不加功能：风险分级漏判、审计脱敏误伤时间戳、`reasoning_content` 回传、工具定义重复注入、子进程密钥隔离，每项写 ADR、补单测、更新 LIMITATIONS，各一个提交；子进程密钥隔离必须在 Phase 13 之前完成。 |
| D10 | N2 的追加授权 | 第 1–5 项完成后，允许在 `e_commerce_33` 任务 0 上再执行 1 次 `workbench agent run`（`openai_compat`、默认预算），只用于确认修复没有破坏链路。已于 2026-09-24 在提交 `f135189` 上执行（WALKTHROUGH §5.3）。 |

## 5. Phase 12.5 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D11 | vLLM tool parser | 启用 `--enable-auto-tool-choice --tool-call-parser hermes`，只改 `configs/serving/arctic-awm-4b.yaml` 与 serve 脚本。在 ADR 或 RECON 中写明依据（Arctic-AWM chat template 的 `<tool_call>{...}</tool_call>` 格式与 vLLM hermes parser 的解析格式一致，引用源码行号），并从 vLLM 源码确认不带 `tools` 参数的请求（`awm agent` 的文本协议）不受 parser 影响。标注 UNVERIFIED-LOCAL，留到 Phase 15 在 GPU 上验证。 |
| D12 | `DEEPSEEK_API_KEY` | 暂不轮换，由仓库主人自己检查 DeepSeek 后台的用量记录。**Phase 13 结束后仓库主人会删除这把 key，Phase 13 之后的阶段不再需要该 key。** Phase 13 报告末尾须提醒这一步。 |
| D13 | ADR-019 遗留的两处隔离缺口 | (a) `awm verify` 经本地代理运行、只拿占位 key；(b) `workbench train launch` 改用白名单。**两项都不在当前阶段做。** 如果仓库主人决定执行 Phase 15，这两项作为 Phase 15 的前置修复（先修再跑）；否则保留在 LIMITATIONS。 |

N5 核对（2026-09-24）：§2 中"本轮结束后从 `phase9-verification` 向 `main` 开 PR"记为已授权、待本轮最后一个阶段结束后执行，与仓库主人的说法一致，无需更正。（`docs/process/TASK_v2.md` N5 中该项的复选框未勾选；以本文件记录的对话授权为准。）

## 6. Phase 13 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D14 | `DEEPSEEK_API_KEY` | 仓库主人删除这把 key；**之后所有阶段不得依赖它**，也不再调用任何付费 API。 |
| D15 | Phase 14（Docker） | 先尝试在容器内启动 `dockerd`。起不来时不必停下等仓库主人，改为在 GitHub Actions 中新增 `docker-smoke` 任务（托管的 ubuntu runner 自带 Docker）：<br>- 触发：push 到 `phase9-verification` 与 `main`；<br>- 步骤：`docker compose build` → `docker compose up -d`（不带 gpu profile）→ 用 mock 后端做一次 API 级冒烟（查询 → 写操作 → 审批 → 完成）→ `docker compose down`；<br>- 用脚本实测镜像大小与冷启动耗时，打印到 job 日志；<br>- 以 CI run 链接和日志摘录作为 U6 的证据，写进 `docs/verification/`；<br>- 只允许为让构建通过而修改 Dockerfile / compose，并补 ADR。 |
| D16 | compose 中的 vllm 服务 | 授权在 `docker-compose.yml` 的 vllm 命令中加上 `--enable-auto-tool-choice --tool-call-parser hermes`，并加单测断言 compose 中 vllm 的参数与 `configs/serving/arctic-awm-4b.yaml` 一致，防止再次漂移。 |
| D17 | 两项零成本工作（不调用任何付费 API） | (a) 用本地假上游代替 DeepSeek，重做"步骤中途中断后续跑"：确认中断确实落在某个 gen 步骤中间，续跑后结果正确；上次信号发错进程组的问题一并修正。<br>(b) 合成预算熔断：本地代理按账本累计费用，超过上限时拒绝转发并让当前步骤失败，提高上限后可以续跑；上限在配置中设置，默认 ¥5；补单测、写 ADR。 |
| D18 | Phase 15 | **要做，采用 runbook 模式**：仓库主人在自己租的 GPU 机器上手动执行，那台机器上不运行 Claude Code。<br>- 先在当前容器完成 D13 的两项前置修复（`awm verify` 经本地代理运行、`train launch` 改用白名单），离线测试通过。<br>- 再写 `docs/runbooks/phase15-gpu.md`：机器要求（显存、CUDA 版本、磁盘）；从零开始的逐条命令及每步的预期输出与判定标准；需要回贴的日志清单与脱敏要求；预计耗时。<br>- 分两段：15A（24GB 显卡）：vLLM 服务 Arctic-AWM-4B，`vllm` 后端的 `workbench agent run`、`awm agent`、`awm verify` 各 1 次；15B（A100 级别）：安装 train 环境、preflight、smoke 训练（不超过 5 step）。<br>- 机器可能在国内（AutoDL），访问 Hugging Face 与 GitHub 受限：runbook 给出用 `HF_ENDPOINT` 从镜像站下载的方式，并标明哪些步骤需要外网。<br>- `awm verify`：从源码确认 `--mode code` 能否不需要 LLM 裁判；需要裁判时指向本机 vLLM 服务，并在 runbook 中注明"裁判为 Arctic-AWM-4B，仅用于打通链路"。<br>- 仓库主人回贴日志后，再更新 U1、U3、U4、U5、U8 与 LIMITATIONS。 |

## 7. Phase 14 报告之后（2026-09-25）

| # | 主题 | 决定 |
|---|---|---|
| D19 | 上游错误让步骤失败（Phase 14 报告的问题 1） | **做，加阈值**，作为 Phase 15 的第三项前置修复：<br>- 代理把"所有重试后仍以上游错误结束"的请求记入账本，状态为 failed；<br>- 新增配置 `synth.max_failed_requests`，默认 0。该步骤 failed 请求数超过阈值时，步骤判失败，可续跑，已付费的请求由缓存重放；超过 0 但不超过阈值时，步骤状态记为 `done_with_failures`，并在报告和 validation 输出中列出失败数，**不得记为 `done`**；<br>- runner 的判定逻辑与现有的 refused 检查统一；<br>- 全部离线测试，零成本：覆盖 402、429、5xx、网络错误四类，以及阈值为 0 和大于 0 两种配置；<br>- 写 ADR，更新 LIMITATIONS 中"步骤是否成功只看退出码"那一条。 |
| D20 | Docker 镜像的分发（只改文档，不改代码） | 在 `docs/UPSTREAM.md` 与 ADR-003 中写明："Docker 镜像内含 AWM 代码，由于 AWM 目前没有许可证，镜像只用于本地和 CI 构建，不得推送到任何公开或私有的镜像仓库"；并检查所有工作流中都没有推送镜像的步骤。 |
| D21 | `DEEPSEEK_API_KEY` 与 Phase 15 的顺序 | 仓库主人自己从环境设置中删除 `DEEPSEEK_API_KEY`。Phase 15 按计划继续：先完成三项前置修复（`awm verify` 经本地代理、`train launch` 白名单、D19），再写 `docs/runbooks/phase15-gpu.md`；**写完 runbook 后停下**，由仓库主人租 GPU 按它执行。 |

## 8. Phase 15 准备完成之后（2026-09-25）

| # | 主题 | 决定 |
|---|---|---|
| D22 | Phase 15 的执行与本轮收尾 | 仓库主人**本轮暂不执行 Phase 15**。（同日较早的一条回贴消息中，说明与两份日志都是未填写的模板，没有据此更新任何验证状态。）按以下方式收尾：<br>- U1、U3、U4、U5、U8 保持未验证，原因写"Phase 15 runbook 已就绪（`docs/runbooks/phase15-gpu.md`），仓库主人尚未在 GPU 机器上执行"；<br>- 按 TASK_v2 §4 完成收尾：LIMITATIONS 分为"已验证"与"未验证"两节，同步更新 README 与 CHANGELOG，按 TASK.md §6 的最终验收清单逐条执行并贴出输出；<br>- 按 N5 从 `phase9-verification` 向 `main` 开 PR，描述列出本轮完成项、仍未验证项及原因、累计费用；<br>- 以后执行 Phase 15 时，仓库主人新开分支单独进行，回贴日志后再提一个小 PR。 |

## 9. polish-v3 本轮开始（2026-09-25）

| # | 主题 | 决定 |
|---|---|---|
| D23 | PR #1 与默认分支 | PR #1 已合并，默认分支已改为 `main`（仓库主人完成）。GitHub 记录 merged_at 2026-09-25T04:41:36Z；`main` 为 `bb8f504`，即 PR #1 的 head，是快进合并，没有单独的 merge commit。从 `main` 新建分支 `polish-v3`，进行 Phase 16–18。 |
| D24 | 本轮规则 | R1–R14 与 N1–N4 全部继续有效；**不调用任何付费 API，LLM 只用 mock；不做任何评测**；工程数字只能由脚本实测，并写明测量命令；每个 Phase 结束停下报告，等"继续"。 |
| D25 | 本轮的 PR | Phase 18 结束后，按 `docs/process/TASK.md` §6 重新执行最终验收清单，然后从 `polish-v3` 向 `main` 开 PR（**已授权**）。 |
