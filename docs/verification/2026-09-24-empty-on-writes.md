# 2026-09-24 ADR-014 的 empty 判定是否误伤写操作

仓库主人在 Phase 12 开始时提出的问题：清空购物车后返回 `{"cart_items": []}` 这类写操作的正常结果，是否会被标成 `empty`？`empty` 标签是否会影响智能体后续的流程判断？

## 1. 结论

- **会误伤，已确认为真实缺陷并修复**：修复前，`is_empty_payload` 不区分读写，写操作的正常返回只要满足"所有列表字段为空、没有嵌套对象"就会被标成 `empty`。
- **对流程的影响**：
  - 智能体代码中没有按 `empty` 分支的硬编码流程（`agent/nodes/observe.py` 只把 `status` 连同完整 `result` 交给模型）；
  - 但 act 提示词写明 `"empty" means nothing matched`（`agent/prompts/act.md:9`），模型可能把一次成功的写操作误读为"没有生效"，进而重试或重新规划；
  - 审计日志（`status` 字段）与 Prometheus 指标（`tool_calls{status}`）会把成功的写操作记为 `empty`；
  - 网关 MCP 前端对 `empty` 返回 `isError=False`，外部 MCP 客户端不会当成错误；
  - "无状态变化"守卫用的是数据库指纹加结果文本（`observe.py:35-37`），写操作改变了数据库，因此不会误触发终止；
  - 审批发生在调用之前，不受影响。
- **修复**：`empty` 只用于 `read` 级工具；`write` / `destructive` 工具调用成功时一律为 `ok`。ADR-014 已补充说明。

## 2. 证据

### 2.1 静态扫描（官方数据集 revision `dde80a0`）

- 方法：对 `gen_envs.jsonl` 中 1000 个环境的 `full_code`，用正则找出 `@app.post/put/patch/delete` 路由及其 `response_model`，再解析该 Pydantic 模型的字段类型：字段类型为 `List[...]` 记为列表，类型为另一个模型或 `Dict` 记为对象，其余记为标量。
- 结果：写类路由共 18374 条，其中 1647 条的返回模型只含列表与标量字段（另有 568 条的返回模型未能解析）。`e_commerce_33` 的 18 条写路由都不属于这一类，所以 Phase 11 的测试没有暴露这个问题。
- 这是近似统计：正则解析可能漏判或误判，只用来说明问题并非个例，不是精确数字。
- 例子：`content_platform_1` 的 `cleanup_watch_later_long_videos` 返回 `{"removed_count": int, "remaining_videos": List[...]}`，删除了若干条但没有剩余时会被判为 `empty`；`social_media_4` 的 `patch_hidden_subreddits` 返回 `{"user_id", "hide_subreddit_ids": List[int], ...}`。

### 2.2 运行时复现

在隔离会话中真实启动官方 `social_media_4`（24 个工具），调用：

```
patch_hidden_subreddits {"remove_subreddit_ids": [1, …, 2000]}
→ isError False, {"user_id": 1, "hide_subreddit_ids": [], "nsfw_blur_enabled": true, "updated_at": "…"}
修复前 normalize → empty
DB diff: user_content_preferences changed [1]
```

命令与输出见 `docs/verification/logs/2026-09-24-empty-on-writes.log`。

## 3. 修复后的验证

- 单元测试：`test_writes_are_never_empty`（`normalize(..., read_only=False)`）、`test_empty_status_only_for_read_tools`（网关按分级传入，读工具的空包装仍为 `empty`，获批的写工具返回同样结构时为 `ok`）。
- `official_data` 测试：`test_successful_write_with_empty_lists_is_not_empty` 在官方 `social_media_4` 上重放上述调用，断言分级为非 `read`、归一结果为 `ok`，且数据库确实改变。
- `make test` → 190 passed；`pytest -m official_data` → 5 passed。
