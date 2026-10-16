# Schema 速查（MVP）

與 [ARCHITECTURE.md](./ARCHITECTURE.md) 一致。類型以 Postgres 為準。

## communities

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| slug | text UNIQUE | |
| name_zh | text | |
| is_active | boolean | MVP 僅一列 true |
| created_at | timestamptz | |

## products

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| sku_code | text | 社區內 UNIQUE |
| name_zh | text | |
| unit_label | text | |
| unit_qty_base | numeric | 預設 1 |
| is_active | boolean | |
| sort_order | int | |
| created_at | timestamptz | |

## stores

| 欄位 | 類型 | 備註 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| name_zh | text | |
| chain_code | text NULL | |
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
| product_id | uuid FK | |
| store_id | uuid FK | |
| user_id | uuid FK | |
| original_price | numeric(12,2) | |
| discount_type | text | none / buy_x_get_y / second_half / pay_more_get_one / buy_x_for_price / percent_off / amount_off |
| related_qty | numeric | |
| related_qty_2 | numeric NULL | |
| valid_until | date | |
| unit_price_effective | numeric(12,4) | 伺服器計算 |
| status | text | pending / published / hidden / expired |
| confirm_count | int | 預設 0 |
| reject_count | int | 預設 0 |
| expired_report_count | int | 預設 0 |
| published_at | timestamptz NULL | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

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
| ccs_item_name | text | |
| store_name | text NULL | |
| price | numeric(12,2) | |
| unit_note | text NULL | |
| observed_on | date | |
| source_ref | text | |
| raw | jsonb NULL | |
| created_at | timestamptz | |

## 關係摘要

- community 1 → N products / stores / users（users 可選綁定 community）
- product + store + user → quotes
- quote 1 → N votes / expired_reports
- product 1 → N ccs_prices（可選對應）
