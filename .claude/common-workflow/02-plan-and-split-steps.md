# Step 02 - Plan, Create Branch, and Split into Execution Steps

## Goal

在需求確認後，先判斷專案類型與任務類型，套用對應的架構規範，再建立 branch 並把整體功能拆成可逐步實作、逐步驗證、逐步 commit 的 plan steps，
並將每個步驟寫入 目前的專案的位置`.claude/agents-plan/[your-feature]/plan-step-x.md`。
只有在使用者確認規劃內容合理後，才可進入 Step 03。

---

## Step 0 — Determine Project Type and Task Type

在開始規劃之前，必須先完成以下兩個判斷：

### 1. 專案類型（Project Type）

可由呼叫此 workflow 時傳入，或根據當前 codebase 結構自動判斷：

| 判斷依據                                                        | 結果               |
| --------------------------------------------------------------- | ------------------ |
| 有 `package.json` + React / Vue / Angular 依賴                  | `frontend`         |
| 有 `requirements.txt` / `pyproject.toml` / `go.mod` / `pom.xml` | `backend`          |
| 兩者皆有（monorepo）                                            | 詢問使用者確認範圍 |

### 2. 任務類型（Task Type）

根據需求描述判斷：

| 信號                                                         | 任務類型       |
| ------------------------------------------------------------ | -------------- |
| 「建立新專案」、「從零開始」、「架構設計」、「scaffold」     | `architecture` |
| 「新增功能」、「實作」、「修改」、「新增 API」、「新增頁面」 | `feature`      |

### 3. 套用對應規範

根據以上判斷，**在規劃步驟時必須參照對應的 command guide**：

| Project Type | Task Type      | 參照規範                                    |
| ------------ | -------------- | ------------------------------------------- |
| `backend`    | `architecture` | `.claude/commands/backend-architecture.md`  |
| `backend`    | `feature`      | `.claude/commands/backend-feature.md`       |
| `frontend`   | `architecture` | `.claude/commands/frontend-architecture.md` |
| `frontend`   | `feature`      | `.claude/commands/frontend-feature.md`      |

> 若無法自動判斷，先詢問使用者再繼續。
> 在 Required Output Format 的 Summary 區塊中，明確標示已套用哪個規範。

---

## Core Responsibilities

1. 根據需求命名本次功能資料夾：
   - `agents-plan/[your-feature]/`

2. 自行建立 branch：
   - branch 名稱需清楚反映本次任務性質
   - 例如：
     - `feature/user-profile-edit`
     - `fix/login-token-refresh`
     - `refactor/case-status-flow`

3. 將需求拆成多個明確步驟，每一步應該：
   - 可單獨實作
   - 可單獨測試
   - 可單獨 review
   - 可單獨 commit
   - 避免一個步驟過大

4. 為每個步驟建立 md 檔，路徑格式：
   - `agents-plan/[your-feature]/plan-step-1.md`
   - `agents-plan/[your-feature]/plan-step-2.md`
   - ...

5. 每個 plan-step 文件需包含：
   - 步驟目標
   - 涉及範圍
   - 需要修改的檔案
   - 實作內容
   - 注意事項
   - 驗收標準
   - 測試重點
   - 完成後預期輸出

---

## Plan Step Document Structure

每個 `plan-step-x.md` 至少應包含以下欄位：

### Title

步驟名稱

### Objective

這一步要完成什麼

### Scope

這一步影響哪些模組 / 頁面 / API / DB / 元件

### Files to Modify

預計修改哪些檔案

### Implementation Details

具體會做哪些事情

### Validation

如何驗證這一步成功

### Risks / Notes

風險、限制、相依性、注意事項

### Done Criteria

這一步完成的標準

---

## Execution Rules

- **先判斷 Project Type 和 Task Type，再切 branch**（見 Step 0）。
- 先切 branch，再寫計劃。
- 不可省略 `agents-plan/[your-feature]/`。
- 拆分步驟要以「可落地、可驗證、可 commit」為原則。
- 每個 plan-step 的 Implementation Details 必須符合對應 command guide 的規範與順序（例如 backend-feature 要求 Entity → DTO → Repository → Domain → Service → Router 的順序）。
- 若某步驟太大，必須再細分。
- 若有前後依賴，需明確標示順序。
- 規劃完成後，要先給使用者確認，不可直接進入實作。

---

## Required Output Format

請回覆使用者：

### 0. Project Context

- Project Type：`frontend` / `backend`
- Task Type：`architecture` / `feature`
- 套用規範：`.claude/commands/[對應規範].md`

### 1. Branch

- 預計 branch 名稱：

### 2. Plan Folder

- `agents-plan/[your-feature]/`

### 3. Step Breakdown

- Step 1:
- Step 2:
- Step 3:
- ...

### 4. Summary

- 哪些先做
- 哪些後做
- 哪些是風險點

### 5. Confirmation

- 請使用者確認拆分是否可行
- 若確認無誤，再進入 Step 03

---

## Exit Criteria

只有在以下條件都成立時才能進入 Step 03：

- branch 已建立
- agents-plan 資料夾已規劃
- 每個 plan-step 文件已建立
- 使用者已確認規劃內容 OK
