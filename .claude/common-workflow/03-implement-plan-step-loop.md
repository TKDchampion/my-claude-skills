# Step 03 - Implement Plan Steps in Loop

## Goal

根據 Step 02 的規劃，逐一實作 `agents-plan/[your-feature]/plan-step-x.md`，
每完成一個步驟後執行 `commands/git-review` 查看 diff 是否滿意，
確認後執行 `commands/git-commit`，然後自動 loop 繼續實作下一個步驟，
直到所有 plan-step 全部完成為止。

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

---

## Loop Execution Flow

對所有 plan-step 依序執行以下 loop，直到全部完成：

```
LOOP over plan-step-1 → plan-step-2 → ... → plan-step-N:

  1. 讀取當前 plan-step-x.md
  2. 選擇對應 commands
  3. 進行實作
  4. 回報本步驟完成內容（見 Required Output Format）

  ── HARD STOP ── 實作完成後必須立即執行以下操作，不可跳過或合併步驟：

  5. 使用 Skill tool 呼叫 `git-review`
     → skill: "git-review"
     → 展示完整 diff 報告，等待使用者明確確認（OK / 需修改）

  6. ── WAIT FOR USER CONFIRMATION ── 禁止自動往下
     → 若使用者說需修改：修正後回到步驟 5，重新執行 git-review
     → 若使用者確認 OK：繼續步驟 7

  7. 使用 Skill tool 呼叫 `git-commit`
     → skill: "git-commit"
     → commit message 必須包含本 plan-step 編號與目標
     → commit 完成後才可繼續

  8. 告知使用者「plan-step-x 已完成並 commit」
     → ── WAIT FOR USER CONFIRMATION ── 等待使用者明確說「繼續」或「ok」後，才進入下一個 plan-step
     → 禁止自動進入下一步

END LOOP when 所有 plan-step 皆完成
```

> **嚴格禁止**：不得在未呼叫 Skill tool 執行 git-review / git-commit 的情況下推進到下一個 plan-step。
> 若使用者對 diff 不滿意，先修正再重新呼叫 git-review skill，直到確認為止。

---

## Command Mapping Rules

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

### Git Review

- **必須使用 Skill tool 呼叫**：`skill: "git-review"`
- 展示完整 diff 報告
- **STOP** — 等待使用者明確回覆確認（不可自動繼續）

### Git Commit

- 使用者確認 OK 後，**必須使用 Skill tool 呼叫**：`skill: "git-commit"`
- commit message 格式：`[plan-step-x] <本步驟目標>`
- commit 成功後，告知使用者並繼續下一個 plan-step

---

## Exit Criteria

只有在以下條件都成立時才能進入 Step 04：

- 所有 plan-step 都已完成實作
- 每個 plan-step 都已執行 git-review 並經使用者確認
- 每個 plan-step 都已執行 git-commit
