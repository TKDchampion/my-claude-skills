# Step 06 - Final Summary, Change Explanation, and Delivery Report

## Goal

整理本次任務從需求、規劃、實作、測試、review 到 security review 的完整結果，
清楚說明每個 step 做了什麼、為什麼這樣做、有哪些變更、目前狀態如何、還有沒有待辦或風險。
確認使用者滿意後，正式完成任務。

---

## Core Responsibilities

1. 回顧所有 plan-step：
   - 每一步做了什麼
   - 修改了哪些檔案
   - 解決了哪些問題
   - 為何這樣設計

2. 彙整所有整體變更：
   - 功能變更
   - 架構變更
   - 資料流 / API / UI / DB 變更
   - 測試結果
   - review 結果
   - security review 結果

3. 說明目前最終狀態：
   - 已完成項目
   - 尚未完成但刻意排除項目
   - 已知限制
   - 建議後續優化方向

4. 最後再次詢問使用者：
   - 是否確認任務完成
   - 是否需要再做一輪 review / security review / optimization

---

## Final Report Structure

### 1. Task Summary

- 本次任務目標
- 最終交付結果

### 2. Step-by-Step Summary

- plan-step-1：
- plan-step-2：
- plan-step-3：
- ...

### 3. Files / Modules Affected

- 主要修改檔案：
- 主要影響模組：

### 4. Testing Summary

- type check：
- lint：
- unit test：
- build：
- manual verification：

### 5. Review Summary

- code review 結果：
- improvement 結果：
- security review 結果：

### 6. Remaining Notes

- 已知限制：
- 後續建議：
- 可再優化項目：

### 7. README.md

在使用者確認任務完成前，必須在專案根目錄產生或更新 `README.md`，內容包含：

- **Project Overview** — 此專案是什麼、解決什麼問題
- **Tech Stack** — 使用的主要技術與框架
- **Project Structure** — 目錄結構說明（`tree` 輸出 + 重要目錄用途）
- **Getting Started** — 環境需求、安裝步驟、啟動指令
- **Key Features** — 本次任務新增 / 修改的主要功能列表
- **API / Routes Reference**（如有）— 主要端點或頁面路由一覽
- **Testing** — 如何執行測試、覆蓋範圍說明
- **Known Limitations / TODOs** — 已知限制與建議後續優化方向

規則：
- 若專案已有 `README.md`，只更新受本次任務影響的區段，不覆蓋無關內容。
- 語言跟隨使用者，若對話為中文則 README 以中文為主（技術術語可保留英文）。
- README 寫完後，告知使用者已更新，並列出修改的區段。

### 8. Final Confirmation

- 請使用者確認是否結案
- 若無問題，正式完成任務
- 若仍需調整，可回到適當步驟繼續處理

---

## Execution Rules

- 最終報告必須清楚、可交付、可回顧。
- 不能只說「完成了」，必須明確解釋。
- 若仍有殘留風險或技術債，需誠實標示。
- 若使用者要求，可重新進行 review / 優化循環。
- README.md 為必要交付物，不可省略。

---

## Exit Criteria

任務完成條件：

- 最終報告已輸出
- README.md 已產生或更新
- 使用者確認任務完成
- 無需再進一步修改
