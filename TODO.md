# TODO — 必讀：開工前先完整讀本檔與 docs/HANDOFF.md、docs/ARCHITECTURE.md

## 清單

- [ ] 選定第一個社區（slug、name_zh）
- [ ] 鎖定約 30 個白名單產品（名、單位、基準量）
- [x] 建立並改名 Netlify site：`hkcompare-2buz` → `hkprice` → `comparehk` → `pricecommunity` → https://pricecommunity.netlify.app （site id `a2a9c1f5-a3e3-4e6f-afbd-3606faa87862`；team `lchbb75`）
- [ ] （可選）日後自訂網域；`hkcompare.netlify.app` 仍被他人硬件站佔用，勿再搶
- [ ] 撰寫 Schema SQL / migration 並套用（見 docs/SCHEMA.md）
- [ ] 實作 API（報價、投票、過期舉報、限流、聲譽、有效單位價）
- [ ] 實作 Frontend（顯示優先級見 ARCHITECTURE）
- [ ] Schema 含 product_variants + quotes 維度欄位（district/sub_district/shop_type/packaging_type/price_type/reported_at/source_type）
- [ ] 上線來源 UI：綠 CCS + 橙／黃街坊證實 chip＋固定文案（藍／紫稍後）
- [ ] 產品頁 Tabs：單件裝／排裝／整箱；multipack 總價+件數→自動單位價；禁止跨包裝混比
- [ ] 容量：每容量獨立產品頁；欄位 capacity_value/unit、normalized_capacity、measure_type、family_id
- [ ] 標準單位價 UI：後端存最細單位（ml／g）；前端算 (售價/normalized)×multiplier；主價正常字＋灰色小字 helper；上線僅 liquid 每100ml＋solid 每100g
- [ ] 篩選：shop_type + packaging_type；上線優先連鎖超市＋街坊藥房
- [ ] 預設有效窗：超市~7日；藥房／臨期 24–48h
- [ ] CCS 格價資訊通資料匯入與產品對應
- [ ] Auth 接線
- [ ] Preview 驗證；**經用戶同意**後再上 production
- [ ] **Deploy／SEO（守則）**：Preview 必須 noindex；禁止 publish preview-branch 做正式主網；`--prod` 前 unset `PREVIEW_NOINDEX`；上線後 curl 核實正式無全站 `X-Robots-Tag: noindex`（見 HANDOFF §8）
- [ ] 每次進度更新 HANDOFF.md + 本檔並 push

## 近期待選／MVP 核心之後（可選）

- [ ] **Phase 1（上線日可選輔助）**：前端本機 Tesseract.js OCR；用戶點「影相報價」才 lazy-load；OCR 只填數字欄，**必須用戶確認才可提交**；近零成本。尚未實作程式碼。

## 較後里程碑（刻意延後）

- [ ] **Phase 2（上線後 1–3 個月，有足夠 logs）**：輕量後端異常偵測（如 Isolation Forest）；只攔明顯離譜報價 → 降權或提高確認門檻；**永不自動刪除**
- [ ] **Phase 3（6+ 個月，僅當下拉折扣不足）**：雲端 LLM 解析複雜促銷文案；每日配額 + circuit breaker；僅後端非同步
- [ ] 來源標示：藍（爬蟲網店）＋紫／紅（商戶自主）
- [ ] 可選 GPS lat/lng；其餘 shop_type／price_type 全面啟用
- [ ] 包裝智能提示（簡單規則）：「6罐裝折合更平」／「單買其實更平」（資料穩後）
- [ ] 「相關容量推薦」卡片（同 family_id；標示最平；簡單規則）
- [ ] 標準單位價：count／length、大包每1kg 顯示

## 刻意不做（MVP / 上線日）

- [ ] ~~上線日 Isolation Forest／後端 ML~~
- [ ] ~~上線日雲端 LLM~~
- [ ] ~~社交 / 個人主頁~~
- [ ] ~~強制圖片上傳~~
- [ ] ~~自訂網域~~（除非用戶另指示）
- [ ] ~~跨 packaging 直接比價（整箱折合 vs 單件混排）~~
- [ ] ~~不同容量塞進同一產品頁 Tabs~~
- [ ] ~~跨物理量綱（g vs ml）自動換算／排名~~
