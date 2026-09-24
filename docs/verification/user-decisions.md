# 仓库主人的决定记录（TASK_v2 本轮）

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
| 本轮结束后从 `phase9-verification` 向 `main` 开 PR | **已授权** | 待本轮最后一个阶段结束后执行。 |
| 在 Snowflake-Labs/agent-world-model 开 issue 询问许可证 | **未授权** | 仓库主人用自己的账号发送（Phase 10 报告之后的决定）；Claude Code 不发送。草稿见 `docs/verification/awm-license-issue-draft.md`，其中的 "#17" 为纯文本（在上游仓库发出时会自动链接到其 PR #17），未改动。 |

## 3. Phase 10 报告之后（2026-09-24）

| # | 主题 | 决定 |
|---|---|---|
| D6 | 默认分支 | 仓库主人自己在 GitHub 设置中改为 `main`。 |
| D7 | AWM 许可证 issue | 仓库主人用自己的账号发；N5 第 3 项保持未授权。 |
| D8 | Phase 12 的前置条件 | Phase 11 完成后如果仍看不到 `DEEPSEEK_API_KEY`，在报告中说明并停下；仓库主人会开新会话继续 Phase 12。 |
