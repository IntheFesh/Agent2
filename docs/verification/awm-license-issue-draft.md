# 草稿：向 Snowflake-Labs/agent-world-model 询问许可证（未发送）

> 状态：**草稿，未发送**。按 TASK_v2 N5，需仓库主人审核措辞并明确授权后才能发出。发出前请确认该仓库此时仍没有 LICENSE、且 PR #17 仍未合并。

---

**Title:** Question: what license applies to the agent-world-model code?

**Body:**

Hi, and thank you for releasing Agent World Model, the AgentWorldModel-1K dataset and the Arctic-AWM checkpoints.

I noticed that the Hugging Face pages state licenses for the dataset (CC-BY-4.0) and the models (Apache-2.0), but this repository does not contain a LICENSE file as of commit `85e322f`, and `pyproject.toml` has no `license` field. Without a license, the default is that all rights are reserved, so it is unclear whether others may use, modify or redistribute the code.

Could you let us know which license you intend for this repository? If it is meant to be open source, adding a LICENSE file (and the `license` field in `pyproject.toml`) would make that explicit. I saw that PR #17 proposes MIT, but since it comes from an external contributor I did not want to assume that reflects your intent.

For context: I am building a small application-layer project that references this repository as a pinned git submodule (no copies of your code, no modifications) and I want to make sure I respect your terms.

Thanks again for the work!

---

说明（不随 issue 发送）：

- 措辞只陈述可核实的事实（HF 页面的许可证、仓库缺少 LICENSE、PR #17 的存在），不催促、不预设答案。
- 不提及本仓库的任何性能数字或评测。
