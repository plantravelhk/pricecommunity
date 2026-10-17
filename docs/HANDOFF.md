# HANDOFF — 香港比較網 / pricecommunity

## AI 交接守則（必讀）

1. **完成已委派工作**再交接；不要留下半成品卻聲稱完成。
2. **Preview → Production** 僅在用戶明確同意後進行；勿擅自切正式環境。
3. 每次有意義進度後，**更新本 HANDOFF 與根目錄 TODO.md**，並推上 GitHub。
4. 發現做不完或受阻項目，**明確列出**（誰阻、差什麼），勿默默跳過。
5. 架構階段**不要**擅自寫應用程式碼或建立 Netlify production site，除非任務明確要求。
6. **標注／說明**：預設**唔加**額外標注、註解或說明。若認為需要標注或說明：**先問用戶**，得同意先加。用戶已特別吩咐或註明例外則可直接加。若同更高優先要求衝突（例如 google-seo 必做嘅 meta／title／H1／FAQ），標注守則**讓路**，該等必做項照做、可簡化標注守則。（skill：AI 交接守則／sand-workflow:ai）

---

## 目前狀態（2026-10-16）

| 項目 | 狀態 |
|------|------|
| 架構文件 | ✅ 已寫：README、ARCHITECTURE、SCHEMA、HANDOFF、TODO、.gitignore |
| AI 輔助路線 | ✅ 2026-10-16 定案：Phase1 本機 OCR → Phase2 異常偵測 → Phase3 LLM（見 ARCHITECTURE）；核心仍是規則 |
| 報價維度（A） | ✅ 2026-10-16：district/sub_district、shop_type、packaging/price_type、reported_at；與核心規則平行 |
| 來源標示（B） | ✅ 2026-10-16：source_type 四色文案；上線綠+橙／黃 |
| 包裝捆綁（C） | ✅ 2026-10-16：同容量 product_variants；Tabs 分包裝；禁止整箱折合 vs 單件混比 |
| 容量分頁（D） | ✅ 2026-10-16：不同容量=獨立產品頁 + family_id；推薦卡稍後 |
| 標準單位價（E） | ✅ 2026-10-16：normalized 存 ml／g；前端 (售價/norm)×multiplier；上線 100ml／100g；主價+灰字 |
| 應用程式碼 | ❌ 無（僅 `public/index.html` noindex placeholder） |
| 資料庫 / Schema SQL | ❌ 未建 |
| Netlify site | ✅ 已建並改名（2026-10-16）— 見下表 |
| 目前 URL | https://pricecommunity.netlify.app |
| 原望 URL | `hkcompare.netlify.app` **仍被佔用**（非本隊） |
| GitHub repo | https://github.com/plantravelhk/pricecommunity |
| 品牌 / Bot 名 | 社會比價 |
| 認證帳號推送 | `plantravelhk` via `gh` |

**Netlify site（2026-10-16；已改名）**：

| 項目 | 值 |
|------|-----|
| 現用名 | `pricecommunity` |
| 公開 URL | https://pricecommunity.netlify.app |
| Site id | `a2a9c1f5-a3e3-4e6f-afbd-3606faa87862`（與改名前相同） |
| Team | `lchbb75` |
| 曾用名 | `hkcompare-2buz` → `hkprice` → `comparehk`（已改名，勿再當正式 URL） |
| 原望子網域 | `hkcompare.netlify.app` — **仍被佔用**；標題「HKCompare · 香港硬件價錢比較」（DIY 硬件），**與本專案無關** |
| Placeholder | `public/index.html`，`noindex`；已上線 |
| SSO / 公開存取 | SSO off（公開可開） |
| 待辦 | 可選自訂網域；勿誤連他人 `hkcompare.netlify.app` |

**本階段範圍**：仍以架構為主。已有 Netlify placeholder（pricecommunity）；無正式 frontend／API、無 Supabase project、無 CCS ingest、**無 OCR 程式碼**。

**AI 輔助路線（2026-10-16 用戶定案）**：核心永遠是純規則 + 投票 + 聲譽。Phase 1（上線日**可選**輔助，非核心路徑）= 前端本機 Tesseract.js OCR（點「影相報價」才 lazy-load；填欄後必須用戶確認才提交）。Phase 2／3 刻意延後以控成本、等有足夠資料再說。詳見 `docs/ARCHITECTURE.md`「輔助能力路線」。

**報價維度／來源／包裝／容量／標準單位價（2026-10-16 用戶定案 A+B+C+D+E）**：
- **A** 空間（18區+商圈）、shop_type、packaging_type／price_type、reported_at；預設有效窗超市~7日、藥房／臨期24–48h；篩選先 shop_type+packaging；可選 GPS 稍後。
- **B** 每則 `source_type`＋色塊固定文案；上線只綠（CCS）+橙／黃（街坊證實）；藍／紫稍後。
- **C** 同容量 `product_variants`；報價存總價、後端算單位價；產品頁 Tabs 單件／排裝／整箱；**永不**跨包裝直接比價；智能提示稍後。
- **D** 不同容量 = **獨立產品頁** + `family_id`；「相關容量推薦」稍後。與 C 共存、勿混淆。
- **E** 後端 `normalized_capacity` 一律最細單位（ml／g）；前端 `standard_unit_price = (sale_price/normalized)×display_multiplier`；主價正常字、標準單位價灰字小字；上線僅 liquid 每100ml + solid 每100g；**禁止**跨 g／ml 自動換算或排名；同 family 各頁各自顯示。
以上與投票／聲譽／過期**平行**，不改核心規則。詳見 ARCHITECTURE §2.11–2.15 與 SCHEMA。

---

## 未完成／開放項目

1. **選定第一個社區**（slug + name_zh）
2. **鎖定約 30 個白名單產品**（名、單位、unit_qty_base）
3. ~~建立 Netlify site~~ → 已改名 **`pricecommunity`** → https://pricecommunity.netlify.app（`hkcompare.netlify.app` 仍被他人硬件站佔用）
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
14. family_id＋normalized_capacity＋上線標準單位價 UI（100ml／100g 灰字）；相關容量推薦卡稍後；count／length 稍後

---

## 給下一任 agent

- 先讀：`TODO.md` → `docs/ARCHITECTURE.md` → `docs/SCHEMA.md`
- 預設分支：`main`
- 工作目錄：`/workspace/pricecommunity/`
- 勿擴大範圍至社交／個人主頁／強制上傳圖；上線日勿實作 Phase 2／3；Phase 1 OCR 為可選輔助且尚未開工
- 實作時遵守 A+B+C+D+E：勿跨 packaging 混比；不同容量分頁；標準單位價勿跨量綱排名；上線來源只綠+橙／黃；與核心規則平行
- Netlify：正式子網域 **https://pricecommunity.netlify.app**（site id `a2a9c1f5-a3e3-4e6f-afbd-3606faa87862`）；**不要**當成已擁有 hkcompare.netlify.app
- 更新本檔「目前狀態」與「開放項目」後再結束回合
