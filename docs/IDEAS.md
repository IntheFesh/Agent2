# IDEAS — 想法池（R13：只记录，不实现）

1. **在训练中接入 AWM 环境**：可以复用 OpenEnv 的 `agent_world_model_env`（BSD-3，已实现 WebSocket 会话和步级奖励映射），把它作为 AgentFly 的资源后端；也可以写一个 AgentFly 的 `@tool` / `@reward` 插件，通过 env-manager 的 HTTP 接口获取隔离环境。两种做法都需要 GPU 环境做验证，而且必须先确认 AWM 的许可证。
2. **`awm_meta` 工具协议模式**：Arctic-AWM 在训练时看到的是 AWM 自己的 `list_tools` / `call_tool` 元函数协议（`awm/core/agent.py:88-127`）。可以给 LLM 客户端增加一种"按 AWM 协议对话"的模式，让线上交互更贴近训练时的分布。这需要改造智能体循环，才能处理元函数调用。
3. **网关的上游连接池**：目前每次工具调用都新建一个 MCP session，与 AWM 的做法一致，可以改为按会话复用连接。
4. **审批按参数范围放行**：例如"单价低于 X 的加购无需审批"。这需要策略 DSL 和配套的审计。
5. **合成流水线的并发上限**：在代理层限制同时转发的请求数，从而收紧预算熔断的在途超支（ADR-023）。预算熔断本身已在 Phase 14 实现。
6. **把 AWM 的 `trajectory.json` 转换成本仓库 trace 格式的离线导入器**，这样 UI 能对比同一任务的两种轨迹。
7. **应用层机制的独立实验设计**（见 ADR-013）：如果将来要回答"网关、审批或守卫是否影响任务完成"，需要在独立仓库里做：固定模型与权重、固定任务集、足够样本、预注册指标、在官方 harness 之外单独报告。本仓库不做。
8. ~~**去掉 act 调用中重复的工具定义**~~：已在 Phase 12.5 实现（ADR-018）。plan 请求仍带完整定义，可以考虑只给 plan 列名称与描述、不列 schema。
9. ~~**DeepSeek 思考模式的 `reasoning_content` 回传**~~：Phase 12.5 在 LLM 层实现（ADR-017）；按仓库主人的决定不加 `extra_body` 开关，也不改智能体状态。
10. **trace 中记录输入 / 输出 token 的拆分**：`llm` 事件目前只有合计值，花费账本只能把全部 token 按输出单价计。记录 `prompt_tokens`、`completion_tokens` 以及服务商返回的缓存命中字段，账本可以更准确。
11. ~~**风险分级结合 HTTP 方法**~~：已在 Phase 12.5 实现（ADR-015）。
12. **`awm verify` 只拿占位 key**：像合成流水线那样经本地代理运行 `awm verify --mode sql`，上游 key 只留在代理所在的进程里。这样 verifier 代码读不到真实 key，只能经代理发起调用，也不需要改上游（见 ADR-019 的限制）。仓库主人决定：若执行 Phase 15，本项与 `train launch` 的白名单作为其前置修复（`user-decisions.md` D13）。
13. **生成代码的进程级沙箱**：env server 与 check_all 以另一个用户或在容器中运行，连本用户可读的文件（`.env`、云凭据）也读不到；ADR-019 只隔离了环境变量。
