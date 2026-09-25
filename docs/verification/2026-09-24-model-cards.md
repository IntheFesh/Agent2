# 2026-09-24 Arctic-AWM 模型卡核对

- 访问时间：2026-09-24 13:18 UTC（curl：`resolve/main/<file>` 与 `api/models/<repo>`）。
- 读取的文件：`README.md`、`config.json`、`tokenizer_config.json`、`generation_config.json`、`chat_template.jinja`。

## 汇总

| 模型 | HF 提交 sha | 许可证（卡片元数据） | 卡片元数据 `base_model` | 卡片正文所说的基座 | 架构与层数 | `max_position_embeddings` |
|---|---|---|---|---|---|---|
| `Snowflake/Arctic-AWM-4B` | `437dfa0e12702901eb41c30e9326a90996549650` | apache-2.0 | Qwen/Qwen3-4B | Qwen3-4B | Qwen3ForCausalLM，hidden 2560，36 层 | 40960 |
| `Snowflake/Arctic-AWM-8B` | `63ebcb9960127e42bf381f325ab68937b383d637` | apache-2.0 | **Qwen/Qwen3-4B**（与正文矛盾） | Qwen3-8B | Qwen3ForCausalLM，hidden 4096，36 层 | 40960 |
| `Snowflake/Arctic-AWM-14B` | `fa3e3b1b4021245e2df94844169e95aced55cf07` | apache-2.0 | **Qwen/Qwen3-4B**（与正文矛盾） | Qwen3-14B | Qwen3ForCausalLM，hidden 5120，40 层 | 40960 |

三张卡的最后修改时间都是 2026-02-11（HF API `lastModified`）。

## 原文摘录

4B 卡片（front matter 与正文）：

> license: apache-2.0
> base_model:
> - Qwen/Qwen3-4B
> **Arctic-AWM-4B** is a multi-turn tool-use agent model trained with agentic reinforcement learning on [Qwen3-4B](https://huggingface.co/Qwen/Qwen3-4B), using the fully synthetic environments from AgentWorldModel-1K.

8B、14B 卡片与 4B 卡片逐行 diff，只有标题行和上面这句正文不同（正文分别写 Qwen3-8B、Qwen3-14B）；front matter 中的 `base_model: - Qwen/Qwen3-4B` 三张卡完全相同。结合 `config.json` 的维度，8B、14B 卡片元数据里的 `base_model` 很可能是复制时遗留的错误。这只是推断，未向上游确认。

## chat template 要点

- 三个模型的 `chat_template.jinja` 字节相同（md5 `da05f6b8a81932c7cf5f26eb545d4417`）。
- 与 `Qwen/Qwen3-4B` 的 chat template 相比，唯一差异在解析历史消息中的 `<think>…</think>` 段：Arctic 版本只在内容以 `<think>` 开头时剥掉前缀，Qwen3 原版取最后一个 `<think>` 之后的内容。
- 工具调用格式与 Qwen3 相同：system 中注入 `<tools>…</tools>`，要求模型输出
  `<tool_call>\n{"name": <function-name>, "arguments": <args-json-object>}\n</tool_call>`；工具结果以 `<tool_response>` 包裹；支持 `enable_thinking`。
- `generation_config.json`：`do_sample`、`temperature`、`top_p`、`top_k` 四项与 `Qwen/Qwen3-4B` 的 `generation_config.json` 逐项相同（2026-09-24 用 curl 读取两者比对）；`tokenizer_config.json` 中 `model_max_length` 为 131072，没有内嵌 chat_template（使用独立的 `chat_template.jinja`）。
- 对 Phase 15 的含义：本仓库客户端从正文解析 `<tool_call>` 的做法与该模板的输出格式一致；是否启用 vLLM 的 `hermes` parser 仍需在真实服务上确认（UNVERIFIED-LOCAL）。
- 2026-09-24 补记：仓库主人决定在 serving profile 中启用 `hermes` parser（`user-decisions.md` D11）。依据与源码行号见 ADR-020 与 RECON "Phase 12.5 报告之后"一节，其中同一 revision 的 `chat_template.jinja` 复查后 md5 仍为 `da05f6b8a81932c7cf5f26eb545d4417`；仍待 Phase 15 在 GPU 上验证。

## "是否就是论文 Table 4 的 AWM 行"

三张卡片都链接了论文（`arxiv.org/abs/2602.10090`），也说明是用 AgentWorldModel-1K 做 RL 训练得到的，但**没有任何一张卡片明说该权重就是论文 Table 4 中 AWM 行所评测的模型**，卡片里也没有列出任何评测数字。因此 registry `models` 段中三个 AWM 模型的身份保持 `verified: false`（推定）。

## 许可证与本仓库用途

三个模型均为 apache-2.0，基座 Qwen3 系列同为 apache-2.0（`Qwen/Qwen3-4B` 卡片元数据 `license: apache-2.0`）。本仓库不分发权重，只在文档与脚本中引用模型名，由 vLLM 在运行时下载。与"公开 GitHub 仓库 + 个人作品展示"的用途不冲突。
