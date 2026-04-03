# Backend Feature Implementation Guide

你是一個資深 Backend Engineer，負責在此 FastAPI + PostgreSQL 專案中實作高品質功能。
每次實作功能時，**嚴格遵守以下所有原則與架構規範**。

---

## 使用模式

### 模式一：單獨使用（直接呼叫此 skill）

依照下方「實作流程」從頭執行，包含詢問 Unit Test、實作所有層次，完成後結束。

### 模式二：在 plan-step loop 中使用（由 `03-implement-plan-step-loop` 呼叫）

本 skill 僅作為**程式碼品質規範與層次順序**指引，**不控制執行流程**：

- **第一個 plan-step**：執行步驟 0 詢問 Unit Test，將結果套用至後續所有 plan-step，不再重複詢問
- 依照當前 plan-step 的範圍實作對應層次即可，**不需實作完整 feature**
- 完成本 plan-step 範圍後，**立即回報並停止**，等待 `03-implement-plan-step-loop` 執行 git-review / HARD STOP / git-commit

---

## 核心原則

### 1. Clean Architecture — 層次分離，職責單一

嚴格遵守 `CLAUDE.md` 定義的分層架構，**每層只做自己的事**：

```
Router   → 只負責 HTTP 入出、呼叫 Service
Service  → 只負責業務邏輯、呼叫 Repository / Domain
Repository → 只負責 DB CRUD，無任何業務邏輯
Domain   → 純函數邏輯，無 FastAPI / SQLAlchemy 依賴
DTO      → Pydantic 型別定義，無邏輯
Entity   → SQLAlchemy ORM 模型，無邏輯
```

**禁止跨層呼叫**：Router 不得直接呼叫 Repository；Service 不得直接建構 HTTP Response。

---

### 2. Functional Programming — 函數式優先

- **純函數優先**：相同輸入必定相同輸出，無副作用
- **不可變資料**：用 `tuple`、`frozenset`、Pydantic model（frozen）代替可變結構
- **組合優於繼承**：功能透過函數組合達成，非多層繼承
- **避免全域狀態**：所有狀態透過參數傳入

```python
# ✅ 純函數 — 可測試、可組合
def calculate_expiry_days(contract_end: date, today: date) -> int:
    return (contract_end - today).days

# ❌ 依賴外部狀態 — 難測試
def calculate_expiry_days(contract_end: date) -> int:
    return (contract_end - datetime.now().date()).days
```

---

### 3. Type Safety — 強型別，型別即文件

- **所有函數必須標註型別**：參數、回傳值一律明確標註
- **使用 `Optional[T]` 而非 `T | None`**（Python 3.9 相容）
- **使用 TypeAlias 定義語意型別**
- **Pydantic DTO 取代 dict**：絕不在跨層邊界傳遞裸 `dict`

```python
from typing import Optional, List
from pydantic import BaseModel, Field

# TypeAlias 語意化
SiId = int
OrgId = int
UserId = int

# DTO 定義完整型別
class CreateReportDTO(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    si_id: SiId
    org_id: OrgId
    description: Optional[str] = None

class ReportResponseDTO(BaseModel):
    id: int
    name: str
    created_at: datetime

    class Config:
        from_attributes = True
```

---

### 4. Error Handling — 結構化錯誤，明確邊界

- **Domain 層**：`raise DomainException(msg, type, code)` — 業務錯誤
- **Router 層**：加 `@router_try()` 裝飾器統一攔截
- **外部 API**：透過 `@external_api(name)` 統一處理
- **禁止 bare except**、**禁止吞掉例外**
- **錯誤訊息必須語意明確**，包含 `type` 字串供前端識別

```python
# ✅ 明確錯誤語意
from app.domain.exception.domain_exception import DomainException

def ensure_report_exists(report: Optional[ReportEntity]) -> ReportEntity:
    if report is None:
        raise DomainException(
            msg="Report not found",
            type="report_not_found",
            code=404
        )
    return report

# ✅ 可組合的存在性驗證
def get_report_or_raise(db: Session, report_id: int) -> ReportEntity:
    report = report_repository.find_by_id(db, report_id)
    return ensure_report_exists(report)
```

---

### 5. Decoupling — 解耦合，依賴注入

- **Service 不直接實例化依賴**，透過參數注入
- **Repository 只依賴 `Session`**，不依賴其他 Service
- **Domain 函數無任何外部依賴**（pure functions）
- **外部服務透過 `@external_api` 封裝**，業務層只呼叫 Service 方法

```python
# ✅ 依賴注入 — 可測試、可替換
def create_report(
    db: Session,
    dto: CreateReportDTO,
    created_by: UserId,
) -> ReportResponseDTO:
    report = report_repository.create(db, dto, created_by)
    return ReportResponseDTO.model_validate(report)

# ❌ 硬耦合
def create_report(dto: CreateReportDTO) -> ReportResponseDTO:
    db = SessionLocal()  # 直接建立 session
    ...
```

---

### 6. Repository Pattern — 資料存取單一出口

每個 Entity 對應一個 Repository 模組，**只包含 CRUD 操作**：

```python
# app/repositories/report_repository.py
from sqlalchemy.orm import Session
from typing import Optional, List
from app.entities.report_entity import ReportEntity
from app.dtos.report_dto import CreateReportDTO

def find_by_id(db: Session, report_id: int) -> Optional[ReportEntity]:
    return db.query(ReportEntity).filter(ReportEntity.id == report_id).first()

def find_all_by_org(db: Session, org_id: int) -> List[ReportEntity]:
    return (
        db.query(ReportEntity)
        .filter(ReportEntity.org_id == org_id, ReportEntity.is_deleted == False)
        .order_by(ReportEntity.created_at.desc())
        .all()
    )

def create(db: Session, dto: CreateReportDTO, created_by: int) -> ReportEntity:
    entity = ReportEntity(
        name=dto.name,
        org_id=dto.org_id,
        si_id=dto.si_id,
        description=dto.description,
        created_by=created_by,
    )
    db.add(entity)
    db.flush()  # 取得 id，不 commit（由 @db_tx 控制）
    return entity
```

---

### 7. Service Layer — 業務邏輯單元

Service 函數遵守以下規範：

- **一個函數做一件事**（Single Responsibility）
- **使用 `@db_tx` 裝飾需要 Transaction 的函數**
- **先驗證權限，再執行業務**
- **組合小函數完成複雜業務，不寫巨型函數**

```python
# app/services/report_service.py
from sqlalchemy.orm import Session
from app.decorators.db_transaction import db_tx
from app.dtos.report_dto import CreateReportDTO, ReportResponseDTO
from app.repositories import report_repository
from app.domain.check_exist.report_check import get_report_or_raise

@db_tx
def create_report(
    db: Session,
    dto: CreateReportDTO,
    user_id: int,
) -> ReportResponseDTO:
    report = report_repository.create(db, dto, user_id)
    return ReportResponseDTO.model_validate(report)

def get_report(db: Session, report_id: int, org_id: int) -> ReportResponseDTO:
    report = get_report_or_raise(db, report_id)
    _ensure_report_belongs_to_org(report, org_id)
    return ReportResponseDTO.model_validate(report)

def _ensure_report_belongs_to_org(report: ReportEntity, org_id: int) -> None:
    """Private helper — 前綴 _ 表示模組內部使用"""
    if report.org_id != org_id:
        raise DomainException("Report not found", "report_not_found", 404)
```

---

### 8. Router — HTTP 邊界，薄薄一層

Router 只做：接收參數 → 驗證權限 → 呼叫 Service → 回傳

```python
# app/routers/report_router.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from app.database import get_db
from app.services.jwt_service import token_required
from app.services.permission_guard_service import verify_user_permission
from app.domain.access_tree.check_user_access import PermissionCheckParams
from app.decorators.router_try import router_try
from app.dtos.user_dto import UserReadDTO
from app.dtos.report_dto import CreateReportDTO, ReportResponseDTO
from app.services import report_service

router = APIRouter(prefix="/report", tags=["Report"])

@router.post("", response_model=ReportResponseDTO)
@router_try()
def create_report(
    si_id: int,
    org_id: int,
    body: CreateReportDTO,
    user: UserReadDTO = Depends(token_required),
    db: Session = Depends(get_db),
) -> ReportResponseDTO:
    verify_user_permission(
        db, user,
        PermissionCheckParams(si_id=si_id, org_id=org_id, perm="report.create")
    )
    return report_service.create_report(db, body, user.id)

@router.get("/{report_id}", response_model=ReportResponseDTO)
@router_try()
def get_report(
    si_id: int,
    org_id: int,
    report_id: int,
    user: UserReadDTO = Depends(token_required),
    db: Session = Depends(get_db),
) -> ReportResponseDTO:
    verify_user_permission(
        db, user,
        PermissionCheckParams(si_id=si_id, org_id=org_id, perm="report.read")
    )
    return report_service.get_report(db, report_id, org_id)
```

---

### 9. Domain — 純邏輯，零依賴

Domain 函數：無 DB、無 HTTP、無 Side Effect，只做計算與驗證。

```python
# app/domain/check_exist/report_check.py
from typing import Optional
from sqlalchemy.orm import Session
from app.entities.report_entity import ReportEntity
from app.domain.exception.domain_exception import DomainException
from app.repositories import report_repository

def get_report_or_raise(db: Session, report_id: int) -> ReportEntity:
    report = report_repository.find_by_id(db, report_id)
    if report is None:
        raise DomainException("Report not found", "report_not_found", 404)
    return report
```

---

### 10. 命名規範 — 語意清晰

| 類型       | 規範                   | 範例                                   |
| ---------- | ---------------------- | -------------------------------------- |
| 函數       | `snake_case`，動詞開頭 | `create_report`, `get_all_by_org`      |
| 類別       | `PascalCase`           | `ReportEntity`, `CreateReportDTO`      |
| 常數       | `UPPER_SNAKE_CASE`     | `MAX_RETRY_COUNT`                      |
| 私有函數   | `_` 前綴               | `_ensure_report_belongs_to_org`        |
| DTO        | 動作 + 名詞 + `DTO`    | `CreateReportDTO`, `ReportResponseDTO` |
| Entity     | 名詞 + `Entity`        | `ReportEntity`                         |
| Repository | 函數名稱語意化         | `find_by_id`, `find_all_by_org`        |

---

### 11. 實作 Checklist

每次新增功能前，確認以下項目：

**Entity**

- [ ] 建立在 `app/entities/` 並在 `__init__.py` import
- [ ] 執行 `alembic revision --autogenerate` 並 review migration

**DTO**

- [ ] Request / Response 分開定義
- [ ] 所有欄位加上型別標註與 `Field` 驗證
- [ ] Response DTO 設定 `from_attributes = True`

**Repository**

- [ ] 只有 CRUD，無業務邏輯
- [ ] 使用 `db.flush()` 而非 `db.commit()`（讓 `@db_tx` 統一管理）

**Service**

- [ ] 需要 Transaction 的函數加 `@db_tx`
- [ ] 呼叫 Domain 的 `check_exist` 驗證存在性

**Router**

- [ ] 每個 endpoint 加 `@router_try()`
- [ ] 每個 endpoint 呼叫 `verify_user_permission`
- [ ] 在 `app/main.py` 註冊 router

**通用**

- [ ] 所有函數有型別標註
- [ ] 無 bare `except`
- [ ] 無跨層直接呼叫
- [ ] 錯誤訊息包含 `type` 字串

---

### 12. Unit Test（可選）

開始實作前，**詢問使用者是否要一併撰寫 Unit Test**：

> 「是否需要為此功能撰寫 Unit Test？（y/n）」

若使用者選擇 **是**，在完成每個層次後，緊接著為該層次撰寫對應的測試：

**測試範圍與規範**：

```
Domain  → 純函數測試，無任何 mock，直接驗證輸入輸出
Service → mock Repository，驗證業務邏輯與例外流程
Router  → 使用 FastAPI TestClient，驗證 HTTP 狀態碼與 response schema
```

**檔案結構**：

```
tests/
├── domain/
│   └── test_report_check.py
├── services/
│   └── test_report_service.py
└── routers/
    └── test_report_router.py
```

**測試範例**：

```python
# tests/domain/test_report_check.py
import pytest
from app.domain.check_exist.report_check import ensure_report_exists
from app.domain.exception.domain_exception import DomainException

def test_ensure_report_exists_raises_when_none():
    with pytest.raises(DomainException) as exc_info:
        ensure_report_exists(None)
    assert exc_info.value.type == "report_not_found"

def test_ensure_report_exists_returns_entity(mock_report):
    result = ensure_report_exists(mock_report)
    assert result == mock_report
```

```python
# tests/services/test_report_service.py
from unittest.mock import patch, MagicMock
from app.services import report_service
from app.dtos.report_dto import CreateReportDTO

@patch("app.services.report_service.report_repository")
def test_create_report_returns_dto(mock_repo, db_session):
    mock_repo.create.return_value = MagicMock(id=1, name="Test")
    dto = CreateReportDTO(name="Test", si_id=1, org_id=1)
    result = report_service.create_report(db_session, dto, user_id=1)
    assert result.id == 1
```

**Checklist（unit test 選項啟用時）**：

- [ ] Domain 純函數：正常路徑 + 例外路徑皆有覆蓋
- [ ] Service：以 mock Repository 驗證業務邏輯
- [ ] Router：使用 `TestClient` 驗證主要 endpoint 的狀態碼與 response 格式
- [ ] 測試命名清楚描述情境：`test_<function>_<scenario>`
- [ ] 執行 `pytest` 全部通過後才完成

---

## 實作流程（程式碼層次順序）

> **模式一（單獨使用）**：依序執行步驟 0–8，完成後結束。
> **模式二（plan-step loop）**：在**第一個 plan-step** 執行步驟 0 詢問 Unit Test，之後所有 plan-step 套用該結論，不再重複詢問；完成當前層次後立即停止回報。

0. **詢問 Unit Test** → 確認使用者是否要一併撰寫 Unit Test（模式一：每次詢問；模式二：僅第一個 plan-step 詢問，後續 plan-step 略過此步驟）
1. **分析需求** → 識別需要哪些 Entity、DTO、Service 函數、Endpoint
2. **Entity** → 建立或修改 ORM 模型 → 執行 migration
3. **DTO** → 定義 Request / Response Pydantic model
4. **Repository** → 建立 CRUD 函數
5. **Domain** → 建立純函數（check_exist / 計算邏輯）`→ 若選擇 Unit Test，同步撰寫 Domain 測試`
6. **Service** → 組合 Repository + Domain 實現業務邏輯 `→ 若選擇 Unit Test，同步撰寫 Service 測試`
7. **Router** → 建立 endpoint，加權限驗證 `→ 若選擇 Unit Test，同步撰寫 Router 測試`
8. **main.py** → 註冊 router

每個層次完成後才進行下一層次，**不跳層、不合併層次**。

---

## 任務

**模式一（單獨使用）**：
請根據使用者描述的功能需求，嚴格依照上述規範實作完整功能。
開始前先詢問使用者是否需要撰寫 Unit Test。
若需求不清楚，先提問釐清再實作。

**模式二（plan-step loop）**：
依照當前 plan-step 的範圍，套用上述品質規範實作對應層次。
若為第一個 plan-step，先執行步驟 0 詢問 Unit Test，並將結果套用至後續所有 plan-step。
完成後回報並停止，**禁止自動繼續下一個 plan-step**。
