# Step 04 - Run Full Tests, Fix Bugs, and Repeat Until All Plan Steps Are Complete

## Goal

針對目前已完成的 plan-step 與整體變更，執行完整測試與驗證，包括 unit test、型別檢查、lint、build、功能驗證等。
若發現 bug 或不符合預期，立即修復，必要時回到前面步驟重新調整，直到所有 plan-step 都完成且測試通過。

---

## Core Responsibilities

1. 對每個已完成的 step 與整體功能執行測試。
2. 測試範圍至少包含：
   - unit test
   - type check
   - lint
   - build
   - integration / flow validation（若適用）
   - 手動功能驗證
   - 錯誤情境驗證
   - 邊界條件驗證

3. 若發現 bug：
   - 先定位原因
   - 修復問題
   - 重新測試
   - 必要時回到 Step 01～03 重新調整需求、計畫或實作

4. 這個步驟不是只跑一次，而是會和 Step 01～04 形成迴圈，直到全部 plan-step 都完成。

---

## Required Test Checklist

依專案情況至少檢查：

### Static Checks

- type check
- lint
- format consistency
- import / dependency correctness

### Runtime Checks

- build 成功
- 頁面 / API / 邏輯正常
- 錯誤處理有效
- loading / empty / error state 正常
- 表單 / 驗證 / 權限 / 邏輯分支正確

### Regression Checks

- 既有功能是否被破壞
- 相依模組是否受影響
- 共用元件 / service / utility 是否仍正常

---

## Bug Fix Loop

若測試失敗，必須遵守以下流程：

1. 說明失敗項目
2. 分析原因
3. 修復問題
4. 重新執行完整測試
5. 若影響原始規劃或需求，回到前面步驟修正
6. 直到測試全部通過

---

## Required Output Format

請以以下格式回覆：

### Test Scope

- 本次測試涵蓋：

### Test Results

- type check：
- lint：
- unit test：
- build：
- manual test：
- edge case：

### Bugs Found

- 問題描述：
- 原因分析：
- 修復方式：

### Current Status

- 是否已全部通過
- 若未全部通過，會繼續修復與重測
- 若全部通過，請求進入 Step 05

---

## Exit Criteria

只有在以下條件都成立時才能進入 Step 05：

- 所有 plan-step 已完成
- 所有相關測試都通過
- 已修正已知 bug
- 沒有阻塞性問題
- 使用者確認可進入 code review / security review
