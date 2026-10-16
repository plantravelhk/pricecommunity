# 香港比較網 — MVP 架構

> 極簡、可實作。單一社區 + 約 30 個白名單產品。無社交、無個人主頁。核心是純規則 + 投票 + 聲譽；AI 僅分階段輔助（見 §7）。

## 1. 範圍限制

| 限制 | 說明 |
|------|------|
| 社區 | **1 個**（例如一個屋苑／商場／地區）；`communities` 表預留擴展，MVP 只啟用一列 |
| 產品 | 白名單約 **30** 個 SKU／品項（由營運預先錄入） |
| 用戶 | 註冊後可報價 + 投票；訪客只讀 |
| 內容 | 報價文字欄位為結構化欄位；不強制圖片 |
| 數據骨幹 | 消委會「格價資訊通」（CCS）作參考價；用戶報價作補充 |
| 後端風格 | 純規則、扁平表、`action_log` 審計 |

## 2. 主表與關係

```
communities (1)
    └── products (N, 每容量一列；同系列以 family_id 連結)
            └── product_variants (N: 同容量 single / multipack / case)
    └── stores (N: district, sub_district, shop_type, optional lat/lng)
    └── users (N) ── quotes (N, 掛 variant + source_type) ── votes (N)
                                              └── expired_reports (N)
ccs_prices ── 對應 product / variant (+ 可選 store/地區)
rate_limits ── 綁 user / IP / action
action_log ── 全表審計
```

### 2.1 `communities`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| slug | text UNIQUE | 例如 `demo-estate` |
| name_zh | text | 顯示名 |
| is_active | bool | MVP 僅一列 `true` |
| created_at | timestamptz | |

### 2.2 `products`（父產品／產品族）

每一**容量**一列產品（例如可口可樂 330ml）；同容量的不同包裝見 `product_variants`。**禁止**把整箱折合單價與單件價直接混比。不同容量用 `family_id` 關聯，**各開獨立產品頁**（見 §2.14）。

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| family_id | uuid NULL | 同品牌／同系列不同容量共用；上線建議填 |
| sku_code | text | 內部代號（**單一容量**一列） |
| name_zh | text | 例如「可口可樂 330ml」— **每容量獨立產品頁** |
| capacity_value | numeric NULL | 輸入容量／重量數值，例如 330、1.25 |
| capacity_unit | text NULL | 輸入單位：`ml` \| `L` \| `g` \| `kg`（及稍後 count／length 用件、米等） |
| normalized_capacity | numeric NULL | **後端一律存最細單位**（液體→ml、重量→g），避免精度問題 |
| measure_type | text NULL | `liquid` \| `solid` \| `count` \| `length`；決定標準單位價展示 |
| unit_label | text | 基準單位標籤，例如「罐」「張」「卷」 |
| unit_qty_base | numeric | 基準數量（計算單位價用），預設 1 |
| is_active | bool | 下架則不可新報價 |
| sort_order | int | |
| created_at | timestamptz | |

同一社區內 `sku_code` UNIQUE。  
**容量 ≠ 包裝捆綁**：不同容量 = 不同 `products` 列／不同頁；同容量多件裝 = `product_variants` Tabs（見 §2.13／§2.14）。

### 2.2b `product_variants`（包裝 SKU／子項）

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| product_id | uuid FK → products | 父產品 |
| packaging_type | text | `single` \| `multipack` \| `case`（單件裝／排裝／整箱） |
| units_per_pack | int | 每包／每箱件數；`single`=1 |
| barcode | text NULL | 可選；掃碼對應 Tab |
| name_zh | text NULL | 可選覆寫顯示名 |
| is_active | bool | |
| created_at | timestamptz | |

同一 `product_id` 下 `(packaging_type, units_per_pack)` 宜 UNIQUE。報價掛在 **variant**，不是只掛父產品。

### 2.3 `stores`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK | |
| name_zh | text | 店鋪名 |
| chain_code | text NULL | 可選連鎖代號 |
| district | text NULL | 香港 18 區（上線可必填於報價表單） |
| sub_district | text NULL | 商圈，例如「沙田市中心」「大圍」 |
| shop_type | text | 見 §2.11；上線優先：`chain_supermarket`、`local_pharmacy` |
| lat | numeric NULL | 可選 GPS；上線日不要求 |
| lng | numeric NULL | 可選 GPS；上線日不要求 |
| is_active | bool | |
| created_at | timestamptz | |

### 2.4 `users`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | 對齊 Auth provider |
| email | text | |
| display_name | text | 顯示用（非公開主頁） |
| reputation | int | **起始 60** |
| ads_off_until | timestamptz NULL | 廣告關閉截止 |
| is_banned | bool | 預設 false |
| created_at | timestamptz | |
| updated_at | timestamptz | |

### 2.5 `quotes`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK | |
| product_id | uuid FK | 父產品（冗餘，方便查詢） |
| product_variant_id | uuid FK → product_variants | **必填**；決定包裝維度 |
| store_id | uuid FK | |
| user_id | uuid FK NULL | 街坊報價者；CCS／爬蟲／商戶來源可 NULL |
| original_price | numeric(12,2) | 該包裝**總價**（港元）；單件即單價 |
| discount_type | text | 見 §3 枚舉 |
| related_qty | numeric | 折扣相關件數（如買 X） |
| related_qty_2 | numeric NULL | 第二參數（如送 Y、加購價） |
| packaging_type | text | 冗餘自 variant：`single` \| `multipack` \| `case` |
| price_type | text NULL | `shelf_normal` \| `near_expiry_clearance` \| `checkout_redeem`；上線可空 |
| district | text NULL | 可冗餘自 store；上線篩選用 |
| sub_district | text NULL | 商圈 |
| shop_type | text NULL | 可冗餘自 store |
| source_type | text | 見 §2.12；上線優先 `ccs_official` + `community_verified` |
| reported_at | timestamptz | 報價／觀測時間 |
| valid_until | date | 有效至；可由 `reported_at` + 預設窗推算 |
| unit_price_effective | numeric(12,4) | **系統計算**＝總價÷件數等，不可由客戶端覆寫 |
| status | text | `pending` / `published` / `hidden` / `expired` |
| confirm_count | int | 預設 0 |
| reject_count | int | 預設 0 |
| expired_report_count | int | 預設 0 |
| published_at | timestamptz NULL | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

上線日：**必填**精簡為 product_variant、store（或 district+shop_type）、original_price、source_type、reported_at、packaging 相關；其餘可 NULL。

### 2.6 `votes`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| quote_id | uuid FK | |
| user_id | uuid FK | |
| vote_type | text | `confirm` \| `reject` |
| created_at | timestamptz | |

**UNIQUE (quote_id, user_id)** — 每用戶每報價一票。不可投自己的報價。

### 2.7 `expired_reports`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| quote_id | uuid FK | |
| user_id | uuid FK | |
| created_at | timestamptz | |

**UNIQUE (quote_id, user_id)**。達到門檻後將 quote 標為 `expired`。

### 2.8 `action_log`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | bigserial PK | |
| user_id | uuid NULL | |
| action | text | 例如 `quote.create`, `vote.confirm`, `quote.expire` |
| entity_type | text | `quote` / `user` / … |
| entity_id | uuid NULL | |
| payload | jsonb | 變更摘要 |
| ip_hash | text NULL | 隱私友善雜湊 |
| created_at | timestamptz | |

### 2.9 `rate_limits`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| bucket_key | text UNIQUE | 例如 `user:{id}:quote` |
| window_start | timestamptz | |
| count | int | |
| updated_at | timestamptz | |

亦可用 Redis；MVP 可用此表或 Postgres 函數。

### 2.10 `ccs_prices`（消委會格價資訊通）

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| product_id | uuid FK NULL | 對應白名單產品；未對上則 NULL |
| ccs_item_name | text | 來源品名 |
| store_name | text NULL | |
| price | numeric(12,2) | |
| unit_note | text NULL | |
| observed_on | date | 資料日期 |
| source_ref | text | 批次／URL／檔名 |
| raw | jsonb NULL | 原始列 |
| created_at | timestamptz | |

CCS 作**顯示參考**，不自動覆寫用戶 `quotes`。

### 2.11 報價維度分類（精簡版）— 2026-10-16 定案

與投票／聲譽／過期**平行**，**不改**核心規則引擎。

| 維度 | 欄位 | 說明 |
|------|------|------|
| 空間／地理 | `district` + `sub_district` | 香港 **18 區** + 商圈（如沙田市中心、大圍）。可選 GPS `lat`/`lng`（掛 `stores`）上線日不要求 |
| 店鋪屬性 | `shop_type` | `chain_supermarket` \| `health_beauty_convenience` \| `local_pharmacy` \| `lifestyle_bargain` \| `wet_market_frozen`。**上線優先**：連鎖超市 + 街坊藥房 |
| 包裝／售賣 | `packaging_type` + 可選 `price_type` | 包裝：`single` \| `multipack` \| `case`。售價類型：`shelf_normal` \| `near_expiry_clearance` \| `checkout_redeem`。**永不**把整箱折合價與單件價直接比較 |
| 時間 | `reported_at` + `valid_until` | 預設有效窗：超市約 **7 日**；藥房／臨期 **24–48 小時**。到期後沿用現有灰顯／隱藏 |

篩選 UI 上線先做：**shop_type** + **packaging_type**。其餘欄位可 NULL，逐步收緊。

### 2.12 價格來源視覺標示 — 2026-10-16 定案

每則報價必有 `source_type`。前端用**色塊 chip + 固定文案**（與投票／聲譽／過期平行，不改核心規則）。

| source_type | 色 | 固定文案 |
|-------------|----|----------|
| `ccs_official` | **綠** | 「官方價，穩陣可信賴」 |
| `crawler_mall` | **藍** | 「網店價，有參考價值」（例：HKTVmall 自建爬蟲） |
| `community_verified` | **橙／黃** | 「街坊報價（X人已證實），現場實時價，可能好平，但去到要睇運氣」；X = `confirm_count` |
| `merchant` | **紫／紅** | 「店家提供特價，可能有限量或附帶條件」 |

**上線優先**：只做 **綠（CCS）** + **橙／黃（街坊已證實）**。藍、紫稍後再開。

### 2.13 包裝與捆綁銷售 — 2026-10-16 定案

同一產品族（例如可口可樂 330ml）**不得**把整箱單位價與單罐價放在同一比較列。

| 層 | 作法 |
|----|------|
| 資料 | 父產品 = 產品族；子包裝 SKU = `product_variants`（`packaging_type`、`units_per_pack`、可選 `barcode`）。報價存**包裝總價** `original_price`；後端按件數算 `unit_price_effective` |
| 顯示 | 產品頁 **Tabs**：單件裝／排裝／整箱 — 每 Tab **只**顯示該包裝的報價與單位價 |
| 掃碼 | 掃到 multipack／case 條碼 → 打開對應 Tab |
| 上線優先 | ① multipack 總價 + 件數輸入 → 自動單位價並按 packaging 分類；② 再做 Tabs 分隔單件 vs 多件；③ 稍後：資料穩後用簡單規則提示「6罐裝折合更平」／「單買其實更平」 |
| 範圍 | 上線只支援常見包裝類型（single / multipack / case） |

### 2.14 容量處理（與捆綁分開）— 2026-10-16 定案

同品牌／同系列的**不同容量**（例如可樂 330ml 罐、500ml 樽、1.25L；洗衣液 500ml／1L／2L）= **各自獨立產品頁**，**不是**同一頁上的 Tabs。

| 要點 | 說明 |
|------|------|
| 頁面 | 每容量一頁；頁內顯示**售價** + **標準單位價**（每 100ml 或每 100g） |
| 關聯 | 後端用 `family_id` 連結同一產品線；上線先有欄位即可 |
| 標準單位價 | 見 §2.15；公式 `(售價 / normalized_capacity) × display_multiplier` |
| 上線範圍 | **標準單位價顯示**（僅液體每100ml + 固體每100g）+ `family_id` + 後端存 `normalized_capacity` |
| 稍後可選 | 「相關容量推薦」卡片：列出同 `family_id` 其他容量的標準單位價、標示邊個最平；**簡單規則**，不做複雜 matching |
| 與捆綁共存 | **同容量** multipack／case → 子包裝 **Tabs**（§2.13）；**不同容量** → **分頁** + `family_id`。兩者可並存 |
| 核心規則 | 與來源標示／投票／聲譽／過期**平行**，不改核心規則 |

### 2.15 標準單位價顯示／換算（精簡版）— 2026-10-16 定案

與 multipack Tabs、容量分頁、來源標示、投票／聲譽／過期**平行**，不改核心規則。

#### 後端儲存

- **一律**將容量／重量正規化為最細單位寫入 `normalized_capacity`：液體 → **ml**；重量 → **g**（由 `capacity_value` + `capacity_unit` 換算，例如 1.25 L → 1250 ml；2 kg → 2000 g）。
- 目的：避免浮點／單位混用精度問題；前端只讀 `normalized_capacity` + `measure_type` 做展示換算。
- 報價「售價」用該包裝下的有效單件價（通常即 `unit_price_effective`；與折扣公式銜接後的每件價）。

#### 前端展示（依產品類型）

| measure_type | 標準單位價標示 | display_multiplier | 備註 |
|--------------|----------------|--------------------|------|
| `liquid` | 每 **100ml** | 100 | **上線必做** |
| `solid`（重量） | 每 **100g** | 100 | **上線必做**；大包米／寵物糧等稍後可改每 **1kg**（multiplier=1000） |
| `count` | 每 **1** 件／張／卷 | 1 | 稍後 |
| `length` | 每 **1** 米 | 1 | 稍後 |

**公式**：

`standard_unit_price = (sale_price / normalized_capacity) × display_multiplier`

例：售價 $6.5、normalized_capacity=330（ml）、liquid → `(6.5/330)×100 ≈ $1.97／100ml`。

#### UI

- **主價**（售價／有效單位價）：正常字級、最醒目。
- **標準單位價**：較小、灰色 helper 文案（例如「約 $1.97／100ml」）。
- **禁止**自動換算或排序去比較**不同物理量綱**（例如 g vs ml）。
- 同 `family_id`、不同容量／單位：各頁各自顯示自己的標準單位價；不做跨量綱排名。

#### 上線範圍

- 只做：`liquid` 每 100ml + `solid` 每 100g。
- 後端存 `normalized_capacity`；**前端計算**展示用標準單位價（可不落 DB 欄）。
- `count`／`length`、每 1kg 大包顯示 → 稍後。

## 3. 有效單位價公式

設：

- `P` = `original_price`（原價／標價）
- `Q` = `related_qty`（主要件數）
- `Q2` = `related_qty_2`（次要件數或加購價等，視類型）
- `B` = `products.unit_qty_base`（通常 1）
- 結果寫入 `unit_price_effective`，單位：港元／基準單位（單件）。若報價為 multipack／case：先以包裝總價代入折扣公式得「每包裝有效價」，再 ÷ `units_per_pack`。比較與排序**只在同一 `packaging_type`（同一 Tab）內**進行。跨容量不混列表；標準單位價（每 100ml／100g）僅作頁內／推薦卡展示。

### 3.1 `discount_type` 枚舉

| 值 | 含義 | 公式（有效「單件」價，再 ÷ B） |
|----|------|--------------------------------|
| `none` | 無折扣 | `P` |
| `buy_x_get_y` | 買 X 送 Y（共 X+Y 件付 X 件價） | `(P * Q) / (Q + Q2)`，其中 Q=`related_qty`=X，Q2=`related_qty_2`=Y |
| `second_half` | 第二件半價（兩件） | `(P + P*0.5) / 2` |
| `pay_more_get_one` | 加錢多一件（付 P+Q2 得 2 件；Q2=加購價） | `(P + Q2) / 2` |
| `buy_x_for_price` | X 件合共 P 元 | `P / Q` |
| `percent_off` | 折扣百分比；Q=折扣％（例如 20 即 8 折） | `P * (1 - Q/100)` |
| `amount_off` | 減 Q 元 | `max(P - Q, 0)` |

實作時以伺服器端單一函數 `compute_unit_price_effective(quote, product)` 為準；客戶端只展示。

**顯示標籤**：當需買滿多件才達到該有效價時，前端標示 **「需買滿 X 件」**（X 依類型：`buy_x_get_y` → X+Y；`second_half` / `pay_more_get_one` → 2；`buy_x_for_price` → Q；其餘 → 1）。

## 4. 核心業務規則

### 4.1 聲譽（reputation）

- 新用戶：**60**
- 建議調整（可於實作微調，須寫入 config）：
  - 報價被足夠 `confirm` 並 `published`：+2～+5
  - 報價被大量 `reject` 或系統隱藏：−5～−10
  - 惡意／spam 經人工或規則：可 `is_banned`
- 高聲譽報價優先排序、較低發布門檻；低聲譽需更多確認（見下）

### 4.2 發布／隱藏門檻（建議預設）

| 報價者 reputation | 發布為 `published` | 隱藏為 `hidden` |
|-------------------|--------------------|-----------------|
| ≥ 80 | confirm ≥ 1 或 建立後短延遲自動 publish（可選） | reject ≥ 3 |
| 60–79 | confirm ≥ 2 | reject ≥ 2 |
| < 60 | confirm ≥ 3 | reject ≥ 2 |

- `pending`：未達發布門檻
- 達隱藏門檻 → `hidden`，列表預設不顯示
- 自己不可投自己

### 4.3 過期

1. **時間**：`valid_until < today` → 自動 `expired`（排程或讀時惰性更新）
2. **預設有效窗**（可由 `reported_at` 推算 `valid_until`，用戶可覆寫）：超市／一般貨架 ≈ **7 日**；藥房或 `near_expiry_clearance` ≈ **24–48 小時**
3. **舉報**：`expired_reports` 達門檻（建議 2）→ `expired`
4. 前端：過期項**灰顯或隱藏**（設定開關；MVP 預設灰顯於「含過期」篩選，預設列表隱藏）

### 4.4 無廣告視窗

- 用戶成功建立一則最終進入 `published` 的報價後：設 `ads_off_until = now() + interval '7 days'`（天數可 config）
- 前端／廣告 SDK 讀 flag：`now() < ads_off_until` → 關閉廣告
- 「成功報價者」= 報價達 `published`，非僅提交 `pending`

### 4.5 速率限制（建議）

| 動作 | 限制 |
|------|------|
| 新建 quote | 每用戶 10／小時；5／日／產品／店 |
| vote / expired_report | 每用戶 60／小時 |
| 同一 quote 重複票 | DB UNIQUE 拒絕 |

超限回傳 429；寫入 `action_log`。

### 4.6 排序（產品詳情／列表）

排序與比價**限定在同一 packaging Tab**（同一 `packaging_type`）內：

1. `status = published` 且未過期優先  
2. 報價者 `reputation` 高 → 前（街坊來源）  
3. `unit_price_effective` 低 → 前（同聲譽帶）  
4. `confirm_count` 高 → 前  
5. `reported_at`／`created_at` 新 → 前  

CCS（`ccs_official`）可用綠色 chip 置於「官方參考」或同一 Tab 列表；**不**與其他 packaging 混排。

## 5. 基本 API Surface

前綴建議 `/api/v1`。全部寫入需 auth（除唯讀）。

| Method | Path | 說明 |
|--------|------|------|
| GET | /products | 社區內白名單產品 |
| GET | /products/:id/quotes | 該產品報價列表（支援 status、sort、packaging_type、shop_type、source_type） |
| GET | /stores | 店鋪列表 |
| GET | /ccs/:productId | CCS 參考價 |
| POST | /quotes | 建立報價（含 variant／packaging、source_type）；伺服器算 `unit_price_effective` |
| POST | /quotes/:id/votes | `{ vote_type: confirm\|reject }` |
| POST | /quotes/:id/expired-report | 舉報過期 |
| GET | /me | 自己的 reputation、ads_off_until |
| POST | /auth/* | 交由 Supabase Auth 或等價 |

錯誤：400 校驗、401 未登入、403 禁止、409 重複票、429 限流。

## 6. 前端顯示優先級

產品詳情：**一容量一頁**。頁內 **Tabs** 僅用於同容量包裝（單件裝／排裝／整箱）— 每 Tab 只顯示該 `packaging_type` 的報價。頁上：主價正常字；標準單位價較小灰色 helper（上線：液體每100ml／固體每100g；見 §2.15）。篩選：`shop_type`、`packaging_type`（上線必做）。「相關容量推薦」卡片**不上線日必做**。

每則報價卡片／列：

1. **來源色塊 chip** + 固定文案（見 §2.12；上線先綠 + 橙／黃）
2. **有效單位價**（最大、最醒目；僅相對同 Tab）
3. 包裝總價 `original_price` + 件數（multipack／case）
4. 折扣條款（人類可讀：由 `discount_type` + qty 組句）
5. **需買滿 X 件**（若 X > 1）
6. 有效期 `valid_until`／`reported_at`
7. 店鋪名、district／商圈、確認／否決數
8. 過期：灰階 +「已過期」；預設列表可隱藏

列表頁：各容量產品名 → 各包裝最低有效價摘要（分列，勿混比）→ 進入該容量詳情。  
無強制圖；可選 emoji／佔位圖即可。掃碼 multipack → 對應 Tab。

## 7. 輔助能力路線（分階段）

> 2026-10-16 定案。核心永遠是**純規則 + 投票 + 聲譽**；AI／ML 只作輔助，不取代門檻與聲譽邏輯。Phase 2–3 刻意延後，以控制成本，並避免在資料不足時上模型。

| 階段 | 時機 | 內容 | 成本／約束 |
|------|------|------|------------|
| **Phase 1** | **上線日**（可選輔助，非核心路徑） | 前端**本機** OCR：Tesseract.js，於用戶手機執行。僅在用戶點「**影相報價**」時 **lazy-load**。OCR 只填入數字相關欄位；**用戶必須確認後才可提交** — **永不自動提交**。 | 近零成本（無後端推論） |
| **Phase 2** | 上線後 **1–3 個月**，且已有足夠 logs | 輕量**後端**異常偵測（例如 Isolation Forest）。只攔截**明顯離譜**的報價：降低權重或提高確認門檻。**永不自動刪除**。 | 需累積行為／報價資料後才做 |
| **Phase 3** | **6+ 個月**，且僅當下拉折扣類型不足以表達促銷 | 雲端 LLM 解析複雜促銷文案。**每日配額 + circuit breaker**；**僅後端非同步**（不阻塞提交主路徑）。 | 嚴格配額，防帳單失控 |

### 上線日明確不做（Phase 1 以外）

- 不上 Isolation Forest／任何後端異常模型
- 不上雲端 LLM／聊天 bot／自動審核模型
- OCR 不得略過用戶確認、不得自動 `POST /quotes`
- 不得以模型結果覆寫 `unit_price_effective` 或繞過聲譽／投票規則

Phase 1 程式碼尚未實作；實作時仍屬「輔助」，手動填表報價必須可用。

## 8. 明確非目標

- 上線日後端 ML、Isolation Forest、雲端 LLM、自動審核模型、聊天 bot（見 §7 分階段）
- 社交 feed、追蹤、公開個人主頁
- 強制／必填產品圖片上傳（影相報價為可選 OCR 輔助，非必填產品圖）
- 多社區並行運營（表可有，產品邏輯只開 1 個）
- 複雜促銷引擎（只支援上表枚舉；不足時才考慮 Phase 3）
- 實時競價、庫存、下單支付
- 自訂網域（MVP 現用 `pricecommunity.netlify.app`；`hkcompare.netlify.app` 仍被他人佔用，勿當己有）
- 跨 packaging 直接比價（整箱折合 vs 單件混排）
- 把不同容量塞進同一產品頁 Tabs（容量必須分頁；family_id 只做關聯／稍後推薦）
- 上線日複雜跨容量 matching／推薦引擎
- 自動跨物理量綱換算或排名（g vs ml 等）
- 上線日 count／length 標準單位價、大包每1kg（欄位可先留）
- 上線日藍（爬蟲網店）／紫紅（商戶）來源標示（欄位可先有，UI 稍後開）

## 9. 設定常數（建議集中 config）

```
REPUTATION_START=60
ADS_OFF_DAYS=7
EXPIRED_REPORT_THRESHOLD=2
RATE_QUOTE_PER_HOUR=10
PUBLISH_THRESHOLDS=...（見 §4.2）
VALIDITY_DAYS_SUPERMARKET=7
VALIDITY_HOURS_PHARMACY_OR_NEAR_EXPIRY=24..48
LAUNCH_SOURCE_TYPES=ccs_official,community_verified
LAUNCH_SHOP_TYPES=chain_supermarket,local_pharmacy
STANDARD_UNIT_BASIS=100  # liquid/solid launch display_multiplier
NORMALIZE_LIQUID_TO=ml
NORMALIZE_SOLID_TO=g
LAUNCH_MEASURE_TYPES=liquid,solid
```

---

文件版本：MVP architecture-only。實作前同步更新 `docs/SCHEMA.md` 與 `TODO.md`。
