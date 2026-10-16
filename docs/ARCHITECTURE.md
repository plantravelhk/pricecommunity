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
    └── products (N, whitelist)
    └── stores (N)
    └── users (N) ── quotes (N) ── votes (N)
                              └── expired_reports (N)
ccs_prices ── 對應 product (+ 可選 store/地區)
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

### 2.2 `products`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK → communities | |
| sku_code | text | 內部代號 |
| name_zh | text | 產品名（繁中） |
| unit_label | text | 例如「支」「包」「100g」 |
| unit_qty_base | numeric | 基準數量（計算單位價用），預設 1 |
| is_active | bool | 下架則不可新報價 |
| sort_order | int | |
| created_at | timestamptz | |

同一社區內 `sku_code` UNIQUE。

### 2.3 `stores`

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | uuid PK | |
| community_id | uuid FK | |
| name_zh | text | 店鋪名 |
| chain_code | text NULL | 可選連鎖代號 |
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
| product_id | uuid FK | |
| store_id | uuid FK | |
| user_id | uuid FK | 報價者 |
| original_price | numeric(12,2) | 標價／原價（港元） |
| discount_type | text | 見 §3 枚舉 |
| related_qty | numeric | 折扣相關件數（如買 X） |
| related_qty_2 | numeric NULL | 第二參數（如送 Y、加購價） |
| valid_until | date | 優惠／價格有效至 |
| unit_price_effective | numeric(12,4) | **系統計算**，不可由客戶端覆寫 |
| status | text | `pending` / `published` / `hidden` / `expired` |
| confirm_count | int | 預設 0 |
| reject_count | int | 預設 0 |
| expired_report_count | int | 預設 0 |
| published_at | timestamptz NULL | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

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

## 3. 有效單位價公式

設：

- `P` = `original_price`（原價／標價）
- `Q` = `related_qty`（主要件數）
- `Q2` = `related_qty_2`（次要件數或加購價等，視類型）
- `B` = `products.unit_qty_base`（通常 1）
- 結果寫入 `unit_price_effective`，單位：港元／基準單位

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
2. **舉報**：`expired_reports` 達門檻（建議 2）→ `expired`
3. 前端：過期項**灰顯或隱藏**（設定開關；MVP 預設灰顯於「含過期」篩選，預設列表隱藏）

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

1. `status = published` 且未過期優先  
2. 報價者 `reputation` 高 → 前  
3. `unit_price_effective` 低 → 前（同聲譽帶）  
4. `confirm_count` 高 → 前  
5. `created_at` 新 → 前  

CCS 參考價可並排顯示，不插入 `quotes` 排序陣列（或置頂「官方參考」區塊）。

## 5. 基本 API Surface

前綴建議 `/api/v1`。全部寫入需 auth（除唯讀）。

| Method | Path | 說明 |
|--------|------|------|
| GET | /products | 社區內白名單產品 |
| GET | /products/:id/quotes | 該產品報價列表（支援 status、sort） |
| GET | /stores | 店鋪列表 |
| GET | /ccs/:productId | CCS 參考價 |
| POST | /quotes | 建立報價；伺服器算 `unit_price_effective` |
| POST | /quotes/:id/votes | `{ vote_type: confirm\|reject }` |
| POST | /quotes/:id/expired-report | 舉報過期 |
| GET | /me | 自己的 reputation、ads_off_until |
| POST | /auth/* | 交由 Supabase Auth 或等價 |

錯誤：400 校驗、401 未登入、403 禁止、409 重複票、429 限流。

## 6. 前端顯示優先級

每則報價卡片／列：

1. **有效單位價**（最大、最醒目）
2. 原價 `original_price`
3. 折扣條款（人類可讀：由 `discount_type` + qty 組句）
4. **需買滿 X 件**（若 X > 1）
5. 有效期 `valid_until`
6. 店鋪名、確認／否決數
7. 過期：灰階 +「已過期」；預設列表可隱藏

列表頁：產品名 → 最低有效價摘要 → 進入詳情。  
無強制圖；可選 emoji／佔位圖即可。

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
- 自訂網域（MVP 用 `hkcompare.netlify.app`）

## 9. 設定常數（建議集中 config）

```
REPUTATION_START=60
ADS_OFF_DAYS=7
EXPIRED_REPORT_THRESHOLD=2
RATE_QUOTE_PER_HOUR=10
PUBLISH_THRESHOLDS=...（見 §4.2）
```

---

文件版本：MVP architecture-only。實作前同步更新 `docs/SCHEMA.md` 與 `TODO.md`。
