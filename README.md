# 香港比較網 / hkcompare

**極簡 MVP：香港社區物價比較（報價 + 投票）**

| 項目 | 內容 |
|------|------|
| 品牌 | 香港比較網 / hkcompare |
| 目前 URL | https://pricecommunity.netlify.app （Netlify 名 `pricecommunity`；暫無自訂網域） |
| 原望 URL | hkcompare.netlify.app — **仍被佔用**（他人 DIY 硬件站「HKCompare · 香港硬件價錢比較」，非本專案） |
| 狀態 | 架構文件 + Netlify placeholder（noindex）；**尚未寫應用程式碼** |
| 架構詳情 | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| Schema | [docs/SCHEMA.md](docs/SCHEMA.md) |
| 交接 | [docs/HANDOFF.md](docs/HANDOFF.md) |
| 待辦 | [TODO.md](TODO.md) |

## 目的

讓用戶為約 30 個白名單產品報價（店鋪、原價、折扣類型、相關件數、有效期），系統計算**有效單位價**；其他用戶可確認／否決／舉報過期。消委會「格價資訊通」作骨幹價，用戶報價作補充。單一社區起步，無社交、無個人主頁。核心是規則 + 投票 + 聲譽；AI 僅作分階段輔助（本機 OCR → 異常偵測 → 雲端 LLM），見架構文件。

## 建議技術棧

| 層 | 建議 |
|----|------|
| 前端 / 託管 | Netlify（靜態或 SSR / Edge Functions） |
| 後端 / DB | Supabase（Postgres）或自管 Postgres |
| Auth | Supabase Auth（email／OAuth 均可後定） |
| 廣告 | SDK flag：近期成功報價者關閉廣告 |

純規則引擎 + 扁平表 + action log。AI 輔助分階段：OCR → 異常偵測 → LLM（核心仍是規則）；不上線日就上 Isolation Forest／LLM。

## 範圍外（Out of scope）

- 上線日後端 ML／雲端 LLM、推薦引擎、聊天機器人（AI 輔助見分階段路線）
- 社交動態、關注、留言串
- 用戶個人公開主頁／個人檔案頁
- 強制上傳產品圖
- 多社區／全港同時上線
- 自訂網域（MVP 先用 Netlify 子網域）
- 支付、會員訂閱、積分商城

詳見 [ARCHITECTURE.md § 明確非目標](docs/ARCHITECTURE.md)。

## 授權與貢獻

私人／公開 repo 由擁有者決定。實作前請先讀 `TODO.md` 與 `docs/HANDOFF.md`。
