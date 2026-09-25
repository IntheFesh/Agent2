# TASK_v3：polish-v3 本轮任务（Phase 16–18）

仓库主人在对话中下发的本轮任务原文，Phase 17 开始时按原样存档；为保持原样，两段原文都放在代码块中。

- 第一部分是本轮的任务书；
- 第二部分是 Phase 16 确认时的补充要求；
- 据此作出的决定记在 `docs/verification/user-decisions.md` §9、§10。

## 一、本轮任务（2026-09-25）

```text
PR #1 已由我以 merge commit 方式合并，默认分支已改为 main。从 main 新建分支 polish-v3，开始新一轮。
TASK.md 的 R1–R14 和 TASK_v2 的 N1–N4 全部继续有效；本轮不调用任何付费 API，LLM 只用 mock；不做任何评测；
工程数字只能由脚本实测，并写明测量命令。每个 Phase 结束停下报告，等我回复"继续"。

Phase 16 — 仓库门面
1. 为本仓库自有代码添加 MIT LICENSE（版权人：Yueyi Li），同步 pyproject 的 license 字段；
   README 增加"许可证"一节，写明只覆盖本仓库自有代码，third_party/、数据集、模型各按各自条款（AWM 目前没有许可证）。
2. 扩展 scripts/demo_ui_check.py：在 mock 演示中保存两张截图
   docs/assets/demo-approval.png（审批卡片）和 docs/assets/demo-diff.png（DB diff），嵌入 README 首屏，
   图注写"mock 演示截图"。
3. README 快速开始中的 <this-repo> 换成真实仓库地址；首屏加上 ci 与 docker-smoke 的状态徽章；
   在首屏用 3 行写清"本仓库做了什么 / 上游提供了什么"，并链接 UPSTREAM.md。
4. 把 TASK.md、TASK_v2.md 移到 docs/process/，更新全部引用；CLAUDE.md 留在根目录。
   新增一个检查 README 与 docs 中相对链接是否有效的脚本，加入 make lint 与 CI。
验收：make lint、make test、make check-numbers 全部通过；链接检查通过；CI 为绿。

Phase 17 — 写操作审批前预演
1. write 或 destructive 调用进入审批之前：
   - 复制会话数据库，由 env-manager 在隔离端口起一个影子环境；
   - 在影子环境上执行同一调用，用现有 diff 能力算出改动，然后回收影子环境；
   - 预演结果附在审批请求上。
2. API 与 UI 的审批卡片显示"将要改动的行"；预演失败时显示失败原因。
   是否强制要求预演，由配置项 approval.require_preview 控制，默认 false。
3. 审批令牌绑定（工具、参数哈希、预演 diff 哈希）。
   批准后真实执行，比对实际 diff 与预演 diff；不一致时在审计中记录 preview_mismatch，并在 UI 标出。
4. 影子环境绝不触碰真实数据库；预演有超时；不修改上游。
5. 测试（mock LLM + 迷你夹具，CPU）：
   - 加购的预演 diff 与真实 diff 一致；
   - destructive 操作的预演；
   - 预演失败路径；
   - 参数与令牌绑定不符时拒绝执行；
   - 影子环境回收后没有残留进程和临时文件。
6. 写 ADR；更新 ARCHITECTURE 时序图、WALKTHROUGH；docker-smoke 增加"审批卡片含预演 diff"的断言。

Phase 18 — 可配置的审批策略
1. 新增 configs/approval_policy.yaml：规则按顺序匹配（工具名通配、风险级别、参数条件：数值比较、枚举成员），
   决策为 auto_approve / require_human / deny；没有命中任何规则时，write 与 destructive 一律 require_human。
2. destructive 永远不能 auto_approve，这一点在代码中硬性保证，不依赖配置。
3. 策略文件加载时做 schema 校验，出现未知字段即报错；
   新增 workbench gateway policy test 命令，对给定的调用演示会命中哪条规则、得到什么决策。
4. 审计与 UI 都记录并显示命中的规则编号。
5. 测试：
   - 规则顺序；
   - 数值条件与枚举条件；
   - destructive 硬性拦截；
   - 未命中时的默认行为；
   - schema 校验失败；
   - 审计中记录了规则编号。
6. 写 ADR，更新 LIMITATIONS 与 WALKTHROUGH。

Phase 18 结束后，按 TASK.md §6 重新执行最终验收清单，然后从 polish-v3 向 main 开 PR（已授权）。
```

## 二、Phase 16 确认时的补充（2026-09-25）

```text
Phase 16 确认。

一、分支清理（授权你执行）
我已在 GitHub 设置里把默认分支改为 main。
1. 先用 git ls-remote --symref origin HEAD 确认默认分支是 main；如果不是，停下告诉我，不要执行第 2、3 步。
2. 删除前核对：git diff origin/main origin/claude/kind-gauss-3clgyp 和 git diff origin/main origin/phase9-verification 都为空。
   claude/kind-gauss-3clgyp 上多出的合并提交 71a7185 来自我误操作的 PR #2，没有内容变化，可以丢弃。
3. 核对通过后，删除远端分支 claude/kind-gauss-3clgyp 和 phase9-verification；没有权限就停下告诉我。
4. 授权：Phase 18 结束且最终验收通过后，由你合并 polish-v3 → main 的 PR（优先 merge commit，保留小提交；
   工具不支持时改用快进推送），合并后删除 polish-v3。

二、对你的三个问题
1. docker-smoke（以及 ci.yml，如果它也逐个列了分支）的触发条件改为：push 到 main + pull_request 目标为 main，
   删除按分支名列出的触发。然后现在就从 polish-v3 向 main 开一个 draft PR，后续每次推送的检查都随 PR 运行。
2. 把本轮任务原文存为 docs/process/TASK_v3.md，加入 skip_files 豁免，同步更新固定豁免清单的单测。
3. Phase 17 的预演失败处理按你的方案执行，另外：
   - approval.require_preview 改为按风险级别配置，默认值为 {write: false, destructive: true}。
     为 true 且预演失败时只允许拒绝，不签发令牌；为 false 且预演失败时仍可批准，
     令牌绑定"preview_unavailable"状态，审计中记录，UI 审批卡片醒目标出"未预演"。
   - 预演 diff 与真实 diff 的比对只在结构层面进行：改动的表，新增、删除、修改的主键，改动的列名。
     时间类字段与自动生成的值只记录、不比对；在 RECON 中写明识别这类列的依据，并在审计中注明忽略了哪些列。
     补一个单测：两次执行只有时间戳不同，不应判为 mismatch。
   - 影子环境从会话当前的数据库启动，不是初始数据库。
   - 用脚本实测预演耗时（启动影子环境、执行、diff、回收各多久），作为工程数字记录，并据此设定预演超时。
   - 重新生成 README 截图时，在现有两张之外加一张整页截图（对话、时间线、带预演 diff 的审批卡片同屏）。

继续 Phase 17。
```
