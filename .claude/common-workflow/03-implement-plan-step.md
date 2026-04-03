# Step 03 - Implement Each Plan Step with Proper Commands

## Goal

根據 Step 02 的規劃，逐一實作 `agents-plan/[your-feature]/plan-step-x.md`，
並依任務類型選擇對應的 commands 執行。
每完成一個步驟，都需要先給使用者確認，再執行 git review / git commit，然後進入下一個步驟。

---

## Core Responsibilities

1. 判斷任務類型：
   - 創建新專案
   - 實作新功能
   - 修復 bug
   - 架構調整 / 重構
   - 前端任務
   - 後端任務

2. 根據任務類型套用對應 commands：
   - Frontend feature → `commands/frontend-feature.md`
   - Backend feature → `commands/backend-feature.md`
   - Frontend architecture → `commands/frontend-architecture.md`
   - Backend architecture → `commands/backend-architecture.md`

3. 嚴格依照 `agents-plan/[your-feature]/plan-step-x.md` 順序逐步執行。

4. 每個 plan-step 實作時必須遵守：
   - clean code
   - 可讀性
   - 一致命名
   - 型別明確
   - interface/type 完整
   - error handling
   - 避免過度耦合
   - 符合既有架構
   - 優先最小可行修改，不做無關變更

5. 每完成一個步驟，都要：
   - 說明本次改了什麼
   - 說明為何這樣改
   - 說明有哪些檔案受影響
   - 等待使用者確認

6. 每個 plan-step 經使用者確認後：
   - 執行 `commands/git-review.md`
   - 確認 diff 合理
   - 執行 `commands/git-commit.md`
   - 單獨 commit 該步驟

---

## Command Mapping Rules

請根據情況選擇 commands：

### Feature Development

- 前端功能 → `commands/frontend-feature.md`
- 後端功能 → `commands/backend-feature.md`

### Architecture / Refactor

- 前端架構 → `commands/frontend-architecture.md`
- 後端架構 → `commands/backend-architecture.md`

### Git Process

- review diff → `commands/git-review.md`
- commit changes → `commands/git-commit.md`

---

## Step-by-Step Execution Flow

對每個 `plan-step-x.md` 必須遵守以下流程：

1. 讀取該 step 計劃
2. 選擇對應 commands
3. 進行實作
4. 回報本步驟完成內容
5. 請使用者確認
6. 若確認 OK：
   - 執行 `commands/git-review.md`
   - 執行 `commands/git-commit.md`
7. 再進入下一個 step

---

## Implementation Quality Rules

實作時必須特別注意：

- 不可硬寫 magic value
- 不可留下明顯暫時性 hack
- 需要完整 type/interface
- 必須有必要的 loading / empty / error handling
- 若有 API / DB / state 更新，必須考慮異常情境
- 不做與本步驟無關的大量重構，除非計劃中有寫
- 若實作中發現需求或計劃有偏差，要先提出再改

---

## Required Output Format

每完成一個步驟後，請用以下格式回覆：

### Current Step

- plan-step-x.md
- 本步驟目標：

### Changes Made

- 修改了哪些檔案
- 實作了哪些內容

### Why

- 為什麼這樣設計 / 實作

### Notes

- 風險
- 後續影響
- 待確認項目

### Confirmation

- 請使用者確認這一步是否 OK
- 若確認 OK，再執行 git review / git commit，進入下一步

---

## Exit Criteria

只有在以下條件都成立時才能進入 Step 04：

- 所有 plan-step 都已完成實作
- 每個 plan-step 都已經過使用者確認
- 每個 plan-step 都已執行 git review 與 git commit
