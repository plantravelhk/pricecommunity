# Schema 速查（MVP）

與 [ARCHITECTURE.md](./ARCHITECTURE.md) 一致。類型以 Postgres 為準。  
2026-10-16 更新：A 維度、B 來源、C 包裝、D 容量分頁、E 標準單位價換算。

## communities

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| slug | text UNIQUE | |
| name_zh | text | |
| is_active | boolean | MVP 僅一列 true |
| created_at | timestamptz | |

## products（每一容量一列／獨立產品頁）

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| family_id | uuid NULL | 同系列不同容量共用 |
| sku_code | text | 社區內 UNIQUE |
| name_zh | text | 含容量，例如「可口可樂 330ml」 |
| capacity_value | numeric NULL | 輸入值 |
| capacity_unit | text NULL | ml \| L \| g \| kg（輸入） |
| normalized_capacity | numeric NULL | **後端最細單位**：液體 ml、重量 g |
| measure_type | text NULL | liquid \| solid \| count \| length |
| unit_label | text | 基準單位（件／張／卷等） |
| unit_qty_base | numeric | 預設 1 |
| is_active | boolean | |
| sort_order | int | |
| created_at | timestamptz | |

標準單位價：`standard_unit_price = (sale_price / normalized_capacity) × display_multiplier`；**前端計算展示**，可不存欄。上線：liquid×100（每100ml）、solid×100（每100g）。  
勿跨 g／ml 自動換算或排名。不同容量 = 不同頁；同容量包裝 = `product_variants`。

## product_variants（同容量包裝子 SKU）

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| product_id | uuid FK → products | |
| packaging_type | text | single \| multipack \| case |
| units_per_pack | int | single=1 |
| barcode | text NULL | 可選 |
| name_zh | text NULL | |
| is_active | boolean | |
| created_at | timestamptz | |
| | UNIQUE(product_id, packaging_type, units_per_pack) 建議 | |

## stores

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| name_zh | text | |
| chain_code | text NULL | |
| district | text NULL | 香港 18 區 |
| sub_district | text NULL | 商圈 |
| shop_type | text | chain_supermarket \| health_beauty_convenience \| local_pharmacy \| lifestyle_bargain \| wet_market_frozen |
| lat | numeric NULL | 上線不要求 |
| lng | numeric NULL | 上線不要求 |
| is_active | boolean | |
| created_at | timestamptz | |

## users

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | 對齊 Auth |
| email | text | |
| display_name | text | |
| reputation | int | 起始 60 |
| ads_off_until | timestamptz NULL | |
| is_banned | boolean | 預設 false |
| created_at | timestamptz | |
| updated_at | timestamptz | |

## quotes

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK | |
| product_id | uuid FK → products | 父產品（冗餘） |
| product_variant_id | uuid FK → product_variants | 必填 |
| store_id | uuid FK | |
| user_id | uuid FK NULL | CCS／爬蟲／商戶可空 |
| original_price | numeric(12,2) | 包裝總價 |
| discount_type | text | none / buy_x_get_y / second_half / pay_more_get_one / buy_x_for_price / percent_off / amount_off |
| related_qty | numeric | |
| related_qty_2 | numeric NULL | |
| packaging_type | text | single \| multipack \| case（冗餘） |
| price_type | text NULL | shelf_normal \| near_expiry_clearance \| checkout_redeem |
| district | text NULL | |
| sub_district | text NULL | |
| shop_type | text NULL | |
| source_type | text | ccs_official \| crawler_mall \| community_verified \| merchant |
| reported_at | timestamptz | |
| valid_until | date | |
| unit_price_effective | numeric(12,4) | 伺服器計算（總價÷件數等） |
| status | text | pending / published / hidden / expired |
| confirm_count | int | 預設 0 |
| reject_count | int | 預設 0 |
| expired_report_count | int | 預設 0 |
| published_at | timestamptz NULL | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

上線必填精簡：variant、store（或 district+shop_type）、original_price、source_type、reported_at、packaging；其餘可 NULL。  
上線來源 UI：`ccs_official` + `community_verified`。上線 shop_type 優先：`chain_supermarket` + `local_pharmacy`。

## votes

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| quote_id | uuid FK → quotes | |
| user_id | uuid FK → users | |
| vote_type | text | confirm \| reject |
| created_at | timestamptz | |
| | UNIQUE(quote_id, user_id) | |

## expired_reports

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| quote_id | uuid FK → quotes | |
| user_id | uuid FK → users | |
| created_at | timestamptz | |
| | UNIQUE(quote_id, user_id) | |

## action_log

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | bigserial PK | |
| user_id | uuid NULL | |
| action | text | |
| entity_type | text | |
| entity_id | uuid NULL | |
| payload | jsonb | |
| ip_hash | text NULL | |
| created_at | timestamptz | |

## rate_limits

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| bucket_key | text UNIQUE | |
| window_start | timestamptz | |
| count | int | |
| updated_at | timestamptz | |

## ccs_prices

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| product_id | uuid FK NULL → products | |
| product_variant_id | uuid FK NULL → product_variants | 可選對包裝 |
| ccs_item_name | text | |
| store_name | text NULL | |
| price | numeric(12,2) | |
| unit_note | text NULL | |
| observed_on | date | |
| source_ref | text | |
| raw | jsonb NULL | |
| created_at | timestamptz | |

## 關係摘要

- community 1 → N products / stores
- products 以 `family_id` 組成系列（不同容量各一頁）
- product 1 → N product_variants（**同容量** packaging）
- product_variant + store (+ user) → quotes（含 source_type、維度欄位）
- quote 1 → N votes / expired_reports
- product／variant 1 → N ccs_prices（可選）
- **比價只在同一 packaging_type 內**；勿混 single 與 case 折合價
- **不同容量勿混同一頁 Tabs**；跨容量僅 family_id＋稍後推薦卡
- 標準單位價見 ARCHITECTURE §2.15；後端只存 normalized_capacity
