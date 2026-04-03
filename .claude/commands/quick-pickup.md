# Quick Project Pickup

你是一個資深工程師，任務是快速理解這個專案的全貌，並產出一份完整的專案介紹報告。

---

## 目標

在最短時間內讀懂這個專案，讓任何新加入的工程師可以透過這份報告，快速掌握：

- 專案目的與背景
- 系統架構與模組劃分
- 使用的技術與框架
- 核心實作方式與關鍵流程
- 需要特別注意的事項、慣例、或坑

---

## 執行步驟

### Step 1 — 收集專案基本資訊

依序讀取以下檔案（有的才讀）：

**通用**
- `CLAUDE.md` — 專案規範與 Claude 指引
- `README.md` / `README.zh.md` — 專案說明
- `.env.example` / `.env.sample` — 環境變數清單

**Backend（Python）**
- `requirements.txt` / `pyproject.toml` / `poetry.lock` — 依賴套件
- `app/main.py` / `main.py` — 應用程式入口
- `alembic.ini` / `alembic/` — DB migration 設定
- `app/` 目錄結構

**Frontend（Node.js）**
- `package.json` — 依賴套件與 scripts
- `tsconfig.json` / `vite.config.ts` / `next.config.js` — 建置設定
- `src/` 目錄結構

**其他**
- `docker-compose.yml` / `Dockerfile` — 容器設定
- `.github/workflows/` — CI/CD 流程
- `Makefile` — 常用指令

---

### Step 2 — 深入分析架構

根據 Step 1 收集到的資訊，進一步探索：

**Backend 專案**
1. 掃描 `app/` 的子目錄結構，識別分層（routers / services / repositories / domain / entities / dtos）
2. 讀取 `app/main.py`，了解 router 註冊順序與 middleware
3. 讀取 2～3 個代表性的 router + service + repository 組合，了解實際業務流程
4. 掃描 `app/entities/` 或 `models/`，了解核心資料模型
5. 查看 `app/domain/` 或 `domain/`，了解業務規則
6. 查看 decorator 或共用工具（`@db_tx`, `@router_try`, `@external_api` 等）

**Frontend 專案**
1. 掃描 `src/` 結構，識別 features / pages / components / hooks / api
2. 讀取路由定義（`App.tsx` / `router.tsx` / `_app.tsx` / `pages/`）
3. 讀取 2～3 個代表性的 feature 目錄，了解 API 層 → Hook → 元件 的實際結構
4. 查看 `src/shared/` 或 `src/common/`，了解共用元件與工具
5. 查看狀態管理設定（Zustand store / Redux slices / context）
6. 查看 HTTP client 設定（axios instance / fetch wrapper）

---

### Step 3 — 產出報告

將分析結果寫入 `.claude/agents-plan/introduction-project.md`：
- 若檔案已存在，更新內容（保留原有結構，更新過時資訊）
- 若不存在，建立新檔案

---

## 報告格式

報告必須包含以下所有章節：

```markdown
# 專案介紹

> 最後更新：{today's date}

## 1. 專案概覽

- **目的**：這個專案是做什麼的，解決什麼問題
- **使用者**：誰在用（內部工具 / 外部用戶 / B2B / B2C）
- **專案類型**：Backend API / Frontend SPA / Fullstack / CLI / ...
- **目前狀態**：開發中 / 維護中 / ...

---

## 2. 技術棧

| 類別 | 技術 |
|------|------|
| 語言 | ... |
| 框架 | ... |
| 資料庫 | ... |
| ORM / Query | ... |
| 驗證 / 型別 | ... |
| 測試 | ... |
| CI/CD | ... |
| 容器 | ... |
| 其他工具 | ... |

---

## 3. 系統架構

### 目錄結構

（列出主要目錄，加上一行說明每個目錄的用途）

```
src/ 或 app/
├── ...    # 說明
├── ...    # 說明
└── ...    # 說明
```

### 分層架構

（描述各層的職責與限制，有幾層寫幾層）

---

## 4. 核心模組說明

（列出 3～7 個最重要的模組 / feature，每個說明：）

### {模組名稱}

- **位置**：`path/to/module/`
- **目的**：做什麼
- **關鍵檔案**：
  - `file.ts` — 說明
  - `file.ts` — 說明
- **實作重點**：這個模組有什麼特別的實作方式或邏輯

---

## 5. 關鍵流程

（描述 2～4 個最重要的端到端流程，例如：登入流程、資料建立流程、匯出流程）

### {流程名稱}

1. ...
2. ...
3. ...

---

## 6. 資料模型

（列出核心 Entity / Table，說明重要欄位與關聯）

### {Entity 名稱}

- **對應 Table**：`table_name`
- **重要欄位**：
  - `field_name`：說明
- **關聯**：belongs to / has many ...

---

## 7. API 概覽（Backend 專案）

（列出主要的 API 群組與路由前綴）

| 前綴 | 說明 | Router 檔案 |
|------|------|-------------|
| `/api/v1/...` | ... | `app/routers/...` |

---

## 8. 環境變數

（列出重要的環境變數，說明用途，不要填真實值）

| 變數名稱 | 用途 | 必填 |
|----------|------|------|
| `DATABASE_URL` | PostgreSQL 連線字串 | ✅ |
| `SECRET_KEY` | JWT 簽名密鑰 | ✅ |
| ... | ... | ... |

---

## 9. 開發指令

（列出最常用的指令）

```bash
# 安裝依賴
...

# 啟動開發環境
...

# 執行測試
...

# 執行 migration
...

# Lint / Type check
...
```

---

## 10. 注意事項 & 常見陷阱

（列出新手容易踩的坑、重要慣例、或需要特別注意的事）

- **{注意點 1}**：說明
- **{注意點 2}**：說明
- **{注意點 3}**：說明

---

## 11. 待釐清 / 技術債

（如果在分析過程中發現不清楚的地方、奇怪的實作、或明顯的技術債，列在這裡）

- ...
```

---

## 執行規則

- **只讀不寫**：分析過程只讀取檔案，不修改任何程式碼
- **誠實標註**：若某些資訊無法從現有檔案推斷，在報告中明確標示「需確認」
- **具體不空泛**：每個描述都要帶具體的路徑、函數名、或範例，不要只寫泛泛的描述
- **注意事項要實用**：Section 10 必須包含真正有用的提醒，不是廢話
- **技術債誠實揭露**：若發現不一致、過時、或奇怪的實作，記錄在 Section 11

---

## 完成後

報告寫入完畢後，回覆使用者：

```
報告已產出至 `.claude/agents-plan/introduction-project.md`

快速摘要：
- 專案類型：...
- 主要技術：...
- 核心模組：...（列 3～5 個）

有任何想深入了解的部分嗎？
```
