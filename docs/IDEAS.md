# IDEAS — 想法池（R13：只记录，不实现）

1. **在训练中接入 AWM 环境**：可以复用 OpenEnv 的 `agent_world_model_env`（BSD-3，已实现 WebSocket 会话和步级奖励映射），把它作为 AgentFly 的资源后端；也可以写一个 AgentFly 的 `@tool` / `@reward` 插件，通过 env-manager 的 HTTP 接口获取隔离环境。两种做法都需要 GPU 环境做验证，而且必须先确认 AWM 的许可证。
2. **`awm_meta` 工具协议模式**：Arctic-AWM 在训练时看到的是 AWM 自己的 `list_tools` / `call_tool` 元函数协议（`awm/core/agent.py:88-127`）。可以给 LLM 客户端增加一种"按 AWM 协议对话"的模式，让线上交互更贴近训练时的分布。这需要改造智能体循环，才能处理元函数调用。
3. **网关的上游连接池**：目前每次工具调用都新建一个 MCP session，与 AWM 的做法一致，可以改为按会话复用连接。
4. **审批按参数范围放行**：例如"单价低于 X 的加购无需审批"。这需要策略 DSL 和配套的审计。
5. **合成流水线的并发和预算上限**：在代理层按账本做预算熔断。
6. **把 AWM 的 `trajectory.json` 转换成本仓库 trace 格式的离线导入器**，这样 UI 能对比同一任务的两种轨迹。
7. **应用层机制的独立实验设计**（见 ADR-013）：如果将来要回答"网关、审批或守卫是否影响任务完成"，需要在独立仓库里做：固定模型与权重、固定任务集、足够样本、预注册指标、在官方 harness 之外单独报告。本仓库不做。
8. **去掉 act 调用中重复的工具定义**：工具定义目前同时出现在 system prompt（`tools_block`）和原生 `tools` 参数中；在 39 个工具的官方 `e_commerce_33` 上，每次 act 调用约 26K token（Phase 12 实测，`docs/verification/2026-09-24-llm-chain.md` §3.1）。可以只保留一处，例如只发原生 `tools`，system prompt 里只列工具名与风险级别。需要改 prompt（提升版本号并记入 CHANGELOG）和 act 节点，并在 mock 与真实端点上重新验证。
9. **DeepSeek 思考模式的 `reasoning_content` 回传**：(a) 在 `LLMSettings` 增加 `extra_body` 配置，用 `{"thinking": {"type": "disabled"}}` 选择非思考模式，只涉及 LLM 层与配置；(b) 让 `ChatResult` 携带 `reasoning_content`，由 act 节点写回 assistant 消息，这会改动智能体状态，超出 TASK_v2 Phase 12 第 4 条允许的范围。两者都需要补单测和 ADR。
10. **trace 中记录输入 / 输出 token 的拆分**：`llm` 事件目前只有合计值，花费账本只能把全部 token 按输出单价计。记录 `prompt_tokens`、`completion_tokens` 以及服务商返回的缓存命中字段，账本可以更准确。
11. ~~**风险分级结合 HTTP 方法**~~：已在 Phase 12.5 实现（ADR-015）。
