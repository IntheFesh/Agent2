# 2026-09-24 迷你夹具与官方 e_commerce_33 的对账

- 官方：`data/awm1k`（revision `dde80a0`）中的 `e_commerce_33`，经 env-manager 真实启动后调用 `list_tools`（清单见 `docs/verification/e_commerce_33-tools.md`）。
- 夹具：`tests/fixtures/awm_mini`（手写的 `mini_e_commerce` 场景）。
- 此前状态：夹具的工具名取自 OpenEnv 对 `e_commerce_33` 的抓取记录，标为"暂定"（RECON §1.6）。

## 1. 工具名对照

| 类别 | 数量 | 工具 |
|---|---|---|
| 一致（两边都有） | 7 | `search_products`、`get_product_by_id`、`list_cart_items`、`add_item_to_cart`、`remove_cart_item`、`list_user_payment_methods`、`delete_user_payment_method` |
| 不一致 | 0 | — |
| 仅夹具有 | 0 | — |
| 仅官方有 | 32 | 其余工具（夹具只保留最小必要子集，不补齐） |

结论：夹具的 7 个工具名全部存在于官方场景中，"工具名暂定"的说明可以撤销。

## 2. 参数对照（修正前）

| 工具 | 修正前的夹具 | 官方 | 结论 |
|---|---|---|---|
| `search_products` | `query` **必填**；`max_price`；`sort_by` 为枚举 `price` / `rating` | 全部可选：`query`、`category_id`、`is_prime_eligible`、`min_average_rating`、`max_price`、`sort_by`（自由字符串，描述中列出 `relevance`、`average_rating`、`review_count`、`sales_rank`、`price_asc`、`price_desc`）、`limit`、`offset` | **不一致** |
| `get_product_by_id` | `product_id`（必填） | `product_id`（必填） | 一致 |
| `list_cart_items` | 无参数 | 无参数 | 一致 |
| `add_item_to_cart` | `product_id`、`quantity`（均必填） | `product_offer_id`、`quantity`（必填），`merge_if_exists`（可选） | **不一致**（官方按 offer 而不是 product 加购） |
| `remove_cart_item` | `cart_item_id`（必填） | `cart_item_id`（必填） | 一致 |
| `list_user_payment_methods` | 无参数 | 无参数 | 一致 |
| `delete_user_payment_method` | `payment_method_id`（必填） | `payment_method_id`（必填） | 一致 |

返回结构也不同：修正前的夹具直接返回列表（例如 `[]`），官方返回包装对象（例如 `{"products": [...], "total": n}`、`{"cart_id": 1, "items": [...]}`、`{"payment_methods": [...]}`、`{"success": true}`、`{"cart_item": {...}}`）。

## 3. 修正内容

- `mini_e_commerce_server.py` 重写（手写，未复制官方代码）：
  - 7 个工具的参数名与必填性与官方一致（夹具的参数是官方参数的子集：`search_products` 只保留 `query`、`max_price`、`sort_by`、`limit`；`add_item_to_cart` 保留全部 3 个）；
  - 顶层返回字段名与官方一致；
  - 表结构改为 `products`、`product_aggregates`、`product_offers`、`carts`、`cart_items`、`payment_methods`（官方的表名子集，列已大幅精简）。
- 样例数据、任务与 verifier 仍为手写。offer ID（11–14）与 product ID（1–4）故意不同，以便在 DB diff 中看出误把 product ID 当 offer ID 的调用。
- mock 脚本（`tests/fixtures/trajectories/*.jsonl`）改用官方参数：`sort_by: average_rating`、`add_item_to_cart` 使用 `product_offer_id`。
- `MANIFEST.json` 新增 `interface_reconciled_with` 段：注明来源数据集、revision、CC-BY-4.0 署名、取自官方的内容以及改动说明。
- 自动化校验：`tests/integration/test_official_data.py::test_official_env_starts_and_fixture_matches_its_interface` 在有官方数据时真实启动两个环境，断言夹具 7 个工具的参数名是官方参数的子集、必填字段与官方完全相同。

## 4. 修正后结果

- 该测试通过（见 `docs/verification/logs/2026-09-24-phase11.log`）。
- 依赖夹具的测试同步更新：网关集成测试原先用 `sort_by` 的非法枚举值触发校验错误，官方 `e_commerce_33` 没有枚举参数，改为用官方实际产生的"类型不符"和"缺少必填字段"两种错误。
