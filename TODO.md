# TODO — 必讀：開工前先完整讀本檔與 docs/HANDOFF.md、docs/ARCHITECTURE.md

## 清單

- [ ] 選定第一個社區（slug、name_zh）
- [ ] 鎖定約 30 個白名單產品（名、單位、基準量）
- [ ] 建立 Netlify site（計劃 URL：hkcompare.netlify.app）
- [ ] 撰寫 Schema SQL / migration 並套用（見 docs/SCHEMA.md）
- [ ] 實作 API（報價、投票、過期舉報、限流、聲譽、有效單位價）
- [ ] 實作 Frontend（顯示優先級見 ARCHITECTURE）
- [ ] CCS 格價資訊通資料匯入與產品對應
- [ ] Auth 接線
- [ ] Preview 驗證；**經用戶同意**後再上 production
- [ ] 每次進度更新 HANDOFF.md + 本檔並 push

## 近期待選／MVP 核心之後（可選）

- [ ] **Phase 1（上線日可選輔助）**：前端本機 Tesseract.js OCR；用戶點「影相報價」才 lazy-load；OCR 只填數字欄，**必須用戶確認才可提交**；近零成本。尚未實作程式碼。

## 較後里程碑（刻意延後）

- [ ] **Phase 2（上線後 1–3 個月，有足夠 logs）**：輕量後端異常偵測（如 Isolation Forest）；只攔明顯離譜報價 → 降權或提高確認門檻；**永不自動刪除**
- [ ] **Phase 3（6+ 個月，僅當下拉折扣不足）**：雲端 LLM 解析複雜促銷文案；每日配額 + circuit breaker；僅後端非同步

## 刻意不做（MVP / 上線日）

- [ ] ~~上線日 Isolation Forest／後端 ML~~
- [ ] ~~上線日雲端 LLM~~
- [ ] ~~社交 / 個人主頁~~
- [ ] ~~強制圖片上傳~~
- [ ] ~~自訂網域~~（除非用戶另指示）
