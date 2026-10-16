# HANDOFF — 香港比較網 / hkcompare

## AI 交接守則（必讀）

1. **完成已委派工作**再交接；不要留下半成品卻聲稱完成。
2. **Preview → Production** 僅在用戶明確同意後進行；勿擅自切正式環境。
3. 每次有意義進度後，**更新本 HANDOFF 與根目錄 TODO.md**，並推上 GitHub。
4. 發現做不完或受阻項目，**明確列出**（誰阻、差什麼），勿默默跳過。
5. 架構階段**不要**擅自寫應用程式碼或建立 Netlify production site，除非任務明確要求。

---

## 目前狀態（2026-10-16）

| 項目 | 狀態 |
|------|------|
| 架構文件 | ✅ 已寫：README、ARCHITECTURE、SCHEMA、HANDOFF、TODO、.gitignore |
| AI 輔助路線 | ✅ 2026-10-16 定案：Phase1 本機 OCR → Phase2 異常偵測 → Phase3 LLM（見 ARCHITECTURE）；核心仍是規則 |
| 報價維度（A） | ✅ 2026-10-16：district/sub_district、shop_type、packaging/price_type、reported_at；與核心規則平行 |
| 來源標示（B） | ✅ 2026-10-16：source_type 四色文案；上線綠+橙／黃 |
| 包裝捆綁（C） | ✅ 2026-10-16：同容量 product_variants；Tabs 分包裝；禁止整箱折合 vs 單件混比 |
| 容量分頁（D） | ✅ 2026-10-16：不同容量=獨立產品頁 + family_id；標準單位價；推薦卡稍後 |
| 應用程式碼 | ❌ 無 |
| 資料庫 / Schema SQL | ❌ 未建 |
| Netlify site | ❌ 未建立 |
| 計劃 URL | `https://hkcompare.netlify.app`（子網域；無自訂網域） |
| GitHub repo | https://github.com/plantravelhk/hkcompare |
| 品牌 / Bot 名 | 香港比較網 |
| 認證帳號推送 | `plantravelhk` via `gh` |

**本階段範圍**：architecture-only。無 frontend、無 API、無 Supabase project、無 CCS ingest、**無 OCR 程式碼**。

**AI 輔助路線（2026-10-16 用戶定案）**：核心永遠是純規則 + 投票 + 聲譽。Phase 1（上線日**可選**輔助，非核心路徑）= 前端本機 Tesseract.js OCR（點「影相報價」才 lazy-load；填欄後必須用戶確認才提交）。Phase 2／3 刻意延後以控成本、等有足夠資料再說。詳見 `docs/ARCHITECTURE.md`「輔助能力路線」。

**報價維度／來源／包裝／容量（2026-10-16 用戶定案 A+B+C+D）**：
- **A** 空間（18區+商圈）、shop_type、packaging_type／price_type、reported_at；預設有效窗超市~7日、藥房／臨期24–48h；篩選先 shop_type+packaging；可選 GPS 稍後。
- **B** 每則 `source_type`＋色塊固定文案；上線只綠（CCS）+橙／黃（街坊證實）；藍／紫稍後。
- **C** 同容量 `product_variants`；報價存總價、後端算單位價；產品頁 Tabs 單件／排裝／整箱；**永不**跨包裝直接比價；智能提示稍後。
- **D** 不同容量 = **獨立產品頁** + `family_id`；顯示標準單位價（每100ml／100g）；「相關容量推薦」稍後簡單規則。與 C 共存、勿混淆。
以上與投票／聲譽／過期**平行**，不改核心規則。詳見 ARCHITECTURE §2.11–2.14 與 SCHEMA。

---

## 未完成／開放項目

1. **選定第一個社區**（slug + name_zh）
2. **鎖定約 30 個白名單產品**（名、單位、unit_qty_base）
3. **建立 Netlify site**，確認 subdomain `hkcompare` 可用；綁定 repo 或手動 deploy（需用戶 OK）
4. **撰寫並套用 Schema SQL**（Postgres／Supabase migration）
5. **實作 API**（quotes、votes、expired-report、products、CCS 讀取、rate limit、聲譽規則）
6. **實作 Frontend**（列表、報價表單、投票、有效價／需買滿 X 件、過期灰顯、ads SDK flag、來源 chip、包裝 Tabs、shop_type+packaging 篩選）
7. **CCS（格價資訊通）資料匯入**流程與對應 `product_id`
8. **Auth** 接線（Supabase Auth 或等價）
9. **Config 常數** 落地（聲譽門檻、ads_off 天數、rate limit）
10. Preview 環境驗證後，**經用戶同意**再上 production
11. （可選，非核心）Phase 1 本機 OCR — MVP 核心完成後再考慮；勿自動提交
12. product_variants + 維度／source_type 欄位落地於 SQL
13. 上線來源 UI（綠+橙／黃）與包裝 Tabs；稍後藍／紫與智能比價提示
14. family_id／標準單位價上線；相關容量推薦卡稍後

---

## 給下一任 agent

- 先讀：`TODO.md` → `docs/ARCHITECTURE.md` → `docs/SCHEMA.md`
- 預設分支：`main`
- 工作目錄：`/workspace/hkcompare/`
- 勿擴大範圍至社交／個人主頁／強制上傳圖；上線日勿實作 Phase 2／3；Phase 1 OCR 為可選輔助且尚未開工
- 實作時遵守 A+B+C+D：勿跨 packaging 混比；不同容量分頁（勿用 Tabs）；上線來源只綠+橙／黃；維度欄位與核心規則平行
- 更新本檔「目前狀態」與「開放項目」後再結束回合
