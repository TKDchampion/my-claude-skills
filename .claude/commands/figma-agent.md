# Figma Agent

當使用者輸入包含 Figma URL 時啟動此 agent，目標是快速將設計稿轉為可融入專案的程式碼。

---

## 觸發條件

輸入訊息中包含 Figma URL（`figma.com/` 或 `figma.com/design/` 格式）。

---

## Step 0 — 設計擷取（Figma MCP Only）

> 僅允許使用 Figma MCP 工具。禁止截圖、爬取、或手動描述替代。

執行前必須先呼叫 `figma:figma-use`，再使用 `use_figma` 系列工具取得：

- 畫面結構（frame、section、layer）
- 元件清單（名稱、variant、state）
- Design token（color、typography、spacing、radius）
- 互動狀態（hover、focus、disabled、loading、error）
- 響應式設定（frame 尺寸、auto-layout、constraint）
- 需匯出的資源（icon、image）

輸出格式：

```
## Figma Design Summary

### Frames
- [frame 名稱與用途]

### Components
- [元件名稱、variant、state]

### Design Tokens
- Colors: ...
- Typography: ...
- Spacing / Radius: ...

### Responsive
- [breakpoints 或 frame 尺寸]

### Interaction States
- [各元件的 state]

### Assets
- [需匯出的圖示或圖片]
```

**輸出後停止，等待使用者確認再繼續。**

---

## Step 1 — 快速分析與確認

根據 Design Summary，快速整理：

- 哪些元件可復用既有 shared components
- 哪些需要新建
- 是否有 API 資料需求（或純靜態 UI）
- 實作範圍（哪些 frame / 元件列入本次）

以條列方式回覆，明確問使用者：**「以上理解正確嗎？確認後我將開始實作。」**

**等待使用者確認再繼續。**

---

## Step 2 — 實作

依以下順序快速實作，每層對應 Figma 擷取結果：

1. **Token 對應**：將 Figma color / spacing 映射到 Tailwind config 或 CSS variable（若尚未存在）
2. **Presentational 元件**：對應 Figma 元件，pure UI，只接受 props
   - 元件名稱盡量與 Figma 元件名稱一致（PascalCase）
   - Figma variant → React prop（e.g., `variant="primary"`）
   - 禁止 hardcode 色碼或 px，必須使用 Tailwind class / token
   - 使用超過一次的元件提升到 `shared/components/`
3. **Page / Section 組裝**：將元件組裝成完整畫面
4. **資料接線**（若有）：接上既有 API hook 或建立簡單的 query hook

實作規則：
- 所有樣式使用 Tailwind CSS，禁止 inline style 或單獨 CSS 檔案
- 禁止跨 feature import，shared 元件放 `shared/components/`

---

## Step 3 — 視覺比對

實作完成後，產出比對表：

| 元件 / 頁面 | Figma Frame | Desktop ✅/❌ | Mobile ✅/❌ | 差異說明 |
|---|---|---|---|---|
| | | | | |

確認項目：
- [ ] 色彩、間距、圓角與 Figma 一致
- [ ] 所有 variant / state（hover、focus、disabled、loading、error）正常
- [ ] 響應式在各 breakpoint 正常
- [ ] 無 overflow、截斷、對齊錯誤

回報比對結果，若有差異說明原因並提出修正建議。

---

## Step 4 — 收尾

- [ ] `npm run type-check` 無 error
- [ ] `npm run lint` 無 error
- [ ] 新增的 shared components 列表
- [ ] 尚未實作的部分（若有，說明原因）

輸出交付摘要後結束。

---

## 核心約束

| 規則 | 說明 |
|---|---|
| 設計資訊來源 | 僅限 Figma MCP，禁止任何替代方式 |
| Figma 預設 workflow | 禁止使用 figma-generate-design / figma-implement-design 自動流程 |
| 設計與實作分離 | Step 0 只取資訊，Step 2 才寫 code |
| 步驟確認 | Step 0、Step 1 完成後必須等待使用者確認 |
