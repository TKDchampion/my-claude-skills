# Frontend Feature Implementation Guide

你是一個資深 Frontend Engineer，負責在此 React + TypeScript 專案中實作高品質功能。
每次實作功能時，**嚴格遵守以下所有原則與架構規範**。

---

## 使用模式

### 模式一：單獨使用（直接呼叫此 skill）

依照下方「實作流程」從頭執行，包含詢問 Unit Test、實作所有層次、Lint/Type check，完成後結束。

### 模式二：在 plan-step loop 中使用（由 `03-implement-plan-step-loop` 呼叫）

本 skill 僅作為**程式碼品質規範與層次順序**指引，**不控制執行流程**：

- **第一個 plan-step**：執行步驟 0 詢問 Unit Test，將結果套用至後續所有 plan-step，不再重複詢問
- 依照當前 plan-step 的範圍實作對應層次即可，**不需實作完整 feature**
- 完成本 plan-step 範圍後，**立即回報並停止**，等待 `03-implement-plan-step-loop` 執行 git-review / HARD STOP / git-commit

---

## 核心原則

### 1. Feature-Based 模組架構 — 自包含，公開最小介面

每個功能模組必須**自包含**所有相關程式碼，**不得直接跨 feature import**：

```
src/features/[feature-name]/
├── api/
│   ├── [feature].api.ts       # API 呼叫，繼承 HttpClient
│   └── [feature].keys.ts      # TanStack Query key factory
├── hooks/
│   ├── use-[feature].ts       # Query / Mutation hooks
│   └── use-[feature]-store.ts # Feature-local Zustand slice（若需要）
├── components/
│   ├── [Feature]Page.tsx      # 頁面入口，只做組裝
│   ├── [Feature]List.tsx      # 純展示，接收 props
│   ├── [Feature]Form.tsx      # 表單，含 zod schema
│   └── [Feature]Card.tsx      # 單一資料展示元件
├── types/
│   └── index.ts               # 此 feature 的所有型別定義
├── constants/
│   └── index.ts               # 此 feature 的常數（enum、config）
└── index.ts                   # Barrel export — 只公開外部需要的東西
```

**禁止**：

- 跨 feature 直接 import 內部模組（應透過 `index.ts` barrel）
- 型別與常數定義混在元件檔案裡
- 在 `index.ts` 以外 export 內部實作細節

---

### 2. Functional Programming — 無副作用，可組合

- **純元件優先**：相同 props → 相同輸出，無隱藏副作用
- **函數組合取代繼承**：用 custom hooks 組合邏輯，不用 HOC 繼承鏈
- **不可變資料**：state 更新一律 spread/map/filter，不直接 mutate
- **避免全域可變狀態**：除了 auth、全局 UI state，一律用 props / hooks 傳遞

```tsx
// ✅ 純函數元件 — 可測試、可預期
interface UserCardProps {
  name: string;
  email: string;
  role: UserRole;
  onEdit: (id: number) => void;
}

function UserCard({ name, email, role, onEdit }: UserCardProps) {
  return (
    <div className="user-card">
      <span>{name}</span>
      <span>{email}</span>
      <Badge role={role} />
      <button onClick={() => onEdit(id)}>編輯</button>
    </div>
  );
}

// ❌ 副作用混在元件邏輯裡
function UserCard({ userId }: { userId: number }) {
  // 直接在元件裡 fetch — 難測試、難複用
  const [user, setUser] = useState(null);
  useEffect(() => { fetch(`/users/${userId}`).then(...); }, [userId]);
  ...
}
```

---

### 3. Props 設計 — 介面清晰，職責明確

每個元件的 props 必須遵守以下原則：

**Props 命名規範**：

| 類型             | 規範                      | 範例                               |
| ---------------- | ------------------------- | ---------------------------------- |
| Boolean props    | `is` / `has` / `can` 前綴 | `isLoading`, `hasError`, `canEdit` |
| Event handler    | `on` 前綴，動詞           | `onSubmit`, `onChange`, `onDelete` |
| Render prop      | `render` 前綴             | `renderEmpty`, `renderActions`     |
| Children variant | `children` 或 `slots`     | `children: ReactNode`              |
| Data props       | 名詞，語意清晰            | `user`, `items`, `selectedId`      |

**Props 設計原則**：

```tsx
// ✅ 職責清晰 — 展示與行為分離
interface DataTableProps<T> {
  // 資料
  data: T[];
  isLoading: boolean;
  error?: ApiError;

  // 行為
  onRowClick?: (row: T) => void;
  onSort?: (key: keyof T, dir: SortDirection) => void;

  // 客製化渲染（Render Props）
  renderActions?: (row: T) => ReactNode;
  renderEmpty?: () => ReactNode;

  // 必要識別
  getRowKey: (row: T) => string | number;
}

// ❌ Props 爆炸 — 超過 8 個純資料 props 考慮拆元件或用 data object
interface BadTableProps {
  userName: string;
  userEmail: string;
  userRole: string;
  userCreatedAt: string;
  // ... 20 more user fields
}
// ✅ 改為傳整個物件
interface GoodUserCardProps {
  user: User;
  onEdit: (id: number) => void;
}
```

**禁止**：

- 未明確型別的 `any` props
- 未使用的 props（ESLint `no-unused-vars` 應捕捉）
- 過度 drilling（超過 3 層傳遞同一 prop → 考慮 context 或 store）

---

### 4. 元件切分原則 — 單一職責，適當粒度

**元件切分時機**：

```
1. 一個元件 > 150 行 JSX → 切分
2. 相同 UI 在 2+ 地方使用 → 抽到 shared/components
3. 邏輯可獨立測試 → 抽到 hooks
4. 呈現與資料獲取混在一起 → 分離成 Container + Presentational
5. 條件渲染 > 3 個分支 → 考慮獨立元件或 strategy pattern
```

**Container / Presentational 分離**：

```tsx
// ✅ Presentational — 只管畫面，接受 props
// features/users/components/UserList.tsx
interface UserListProps {
  users: User[];
  isLoading: boolean;
  error?: ApiError;
  onDelete: (id: number) => void;
}

export function UserList({ users, isLoading, error, onDelete }: UserListProps) {
  if (isLoading) return <Skeleton />;
  if (error) return <ErrorState message={error.msg} />;
  if (!users.length) return <EmptyState />;

  return (
    <ul>
      {users.map((user) => (
        <UserCard key={user.id} user={user} onDelete={onDelete} />
      ))}
    </ul>
  );
}

// ✅ Page (Container) — 只管資料獲取與組裝
// features/users/components/UsersPage.tsx
export function UsersPage() {
  const { data: users, isLoading, error } = useUsers();
  const { mutate: deleteUser } = useDeleteUser();

  return (
    <PageLayout title="用戶管理">
      <UserList
        users={users ?? []}
        isLoading={isLoading}
        error={error as ApiError}
        onDelete={deleteUser}
      />
    </PageLayout>
  );
}
```

---

### 5. 型別系統 — 強型別，型別即文件

**型別必須放在 `types/index.ts`，不得散落在元件檔案裡**：

```typescript
// features/users/types/index.ts

// DTO — 對應 API response/request
export interface User {
  id: number;
  name: string;
  email: string;
  role: UserRole;
  createdAt: string;
}

export interface CreateUserDTO {
  name: string;
  email: string;
  roleId: number;
}

export interface UpdateUserDTO extends Partial<CreateUserDTO> {}

// Enum — 語意化狀態
export enum UserRole {
  Admin = "admin",
  Editor = "editor",
  Viewer = "viewer",
}

export enum UserStatus {
  Active = "active",
  Inactive = "inactive",
}

// UI 相關型別
export type SortDirection = "asc" | "desc";

export interface UserFilters {
  search?: string;
  role?: UserRole;
  status?: UserStatus;
}

// Type guards
export function isUserRole(value: string): value is UserRole {
  return Object.values(UserRole).includes(value as UserRole);
}
```

**TypeScript 強制規則**：

- 所有函數參數與回傳值標註型別（`strict: true`）
- 禁止 `any`，用 `unknown` + type narrowing
- API response 必須對應定義好的 interface
- 用 `Readonly<T>` 標記不應被修改的 props/state
- Generic 元件需明確標註 `<T>` 約束

---

### 6. 常數管理 — 集中存放，不散落

**所有常數放在 `constants/index.ts`**：

```typescript
// features/users/constants/index.ts

export const USER_PAGE_SIZE = 20;

export const USER_ROLE_LABELS: Record<UserRole, string> = {
  [UserRole.Admin]: "管理員",
  [UserRole.Editor]: "編輯者",
  [UserRole.Viewer]: "檢視者",
};

export const USER_STATUS_LABELS: Record<UserStatus, string> = {
  [UserStatus.Active]: "啟用",
  [UserStatus.Inactive]: "停用",
};

export const USER_ROLE_OPTIONS = Object.entries(USER_ROLE_LABELS).map(
  ([value, label]) => ({ value, label }),
);

// Query key 也是常數 — 放在 api/users.keys.ts
```

**禁止**：

- magic string / magic number 直接寫在元件裡
- label 對應表散落在各元件
- 重複定義同一個設定值

---

### 7. Hooks — 邏輯封裝，可組合

**每個 custom hook 做一件事**：

```typescript
// features/users/hooks/use-users.ts
import { useQuery } from "@tanstack/react-query";
import { usersApi } from "../api/users.api";
import { usersKeys } from "../api/users.keys";
import type { UserFilters } from "../types";

// 單一職責 hook
export function useUsers(filters?: UserFilters) {
  return useQuery({
    queryKey: usersKeys.list(filters),
    queryFn: () => usersApi.getAll(filters),
  });
}

export function useUser(id: number) {
  return useQuery({
    queryKey: usersKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: !!id,
  });
}

export function useCreateUser() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (dto: CreateUserDTO) => usersApi.create(dto),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: usersKeys.lists() });
      toast.success("用戶建立成功");
    },
  });
}

// features/users/hooks/use-user-filters.ts — 獨立的 UI state hook
export function useUserFilters() {
  const [filters, setFilters] = useState<UserFilters>({});

  const setSearch = useCallback(
    (search: string) => setFilters((prev) => ({ ...prev, search })),
    [],
  );

  const setRole = useCallback(
    (role: UserRole | undefined) => setFilters((prev) => ({ ...prev, role })),
    [],
  );

  const reset = useCallback(() => setFilters({}), []);

  return { filters, setSearch, setRole, reset };
}
```

**Hook 規則**：

- Hook 名稱一律 `use` 開頭
- 只在 React 元件或其他 hooks 頂層呼叫
- 一個 hook 只負責一個關注點（資料獲取 / UI state / 業務邏輯分開）
- 回傳型別明確標註

---

### 8. 表單處理 — React Hook Form + Zod

**Schema 獨立定義，不寫在元件裡**：

```typescript
// features/users/components/UserForm.schema.ts
import { z } from "zod";
import { UserRole } from "../types";

export const createUserSchema = z.object({
  name: z.string().min(1, "名稱不可為空").max(100),
  email: z.string().email("請輸入有效的 Email"),
  roleId: z.nativeEnum(UserRole, {
    errorMap: () => ({ message: "請選擇角色" }),
  }),
});

export type CreateUserFormValues = z.infer<typeof createUserSchema>;
```

```tsx
// features/users/components/UserForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { createUserSchema, type CreateUserFormValues } from "./UserForm.schema";

interface UserFormProps {
  defaultValues?: Partial<CreateUserFormValues>;
  onSubmit: (values: CreateUserFormValues) => void;
  isSubmitting: boolean;
}

export function UserForm({
  defaultValues,
  onSubmit,
  isSubmitting,
}: UserFormProps) {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<CreateUserFormValues>({
    resolver: zodResolver(createUserSchema),
    defaultValues,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <FormField label="姓名" error={errors.name?.message}>
        <input {...register("name")} />
      </FormField>
      <FormField label="Email" error={errors.email?.message}>
        <input type="email" {...register("email")} />
      </FormField>
      <FormField label="角色" error={errors.roleId?.message}>
        <RoleSelect {...register("roleId")} />
      </FormField>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "送出中..." : "送出"}
      </button>
    </form>
  );
}
```

---

### 9. 錯誤處理 — 分層攔截，不遺漏

**四層錯誤處理**：

```
Layer 1: Axios interceptor     → normalize to ApiError, handle token refresh
Layer 2: handleError()         → global toast + auth redirect（HttpClient 預設）
Layer 3: onError in hooks      → feature-specific error handling（override 全局）
Layer 4: GlobalErrorBoundary   → 元件 render crash 的最後防線
```

```tsx
// ✅ Feature-level error override
export function useCreateUser() {
  return useMutation({
    mutationFn: usersApi.create,
    onError: (error: ApiError) => {
      // 覆蓋全局 handler，給更具體的提示
      if (error.type === "email_exists") {
        toast.error("此 Email 已被使用");
      } else {
        handleError(error); // fallback to global
      }
    },
  });
}

// ✅ 元件層面的錯誤邊界（per-feature）
export function UsersPage() {
  return (
    <ErrorBoundary fallback={<FeatureErrorFallback />}>
      <UserListContainer />
    </ErrorBoundary>
  );
}
```

**禁止**：

- `try/catch` 吞掉 error 不處理
- 直接 `console.error` 而不 toast / rethrow
- bare `catch (e) { ... }` 不標註型別

---

### 10. 效能優化 — 精準重渲染，按需載入

**必要時才優化，不過早優化**：

```tsx
// ✅ useMemo — 昂貴計算才包
const filteredUsers = useMemo(
  () => users.filter((u) => u.role === selectedRole),
  [users, selectedRole]
);

// ✅ useCallback — 傳給子元件的 handler 才包（避免無謂 re-render）
const handleDelete = useCallback(
  (id: number) => deleteUser(id),
  [deleteUser]
);

// ✅ React.memo — 純展示元件，props 穩定才有意義
const UserCard = React.memo(function UserCard({ user, onEdit }: UserCardProps) {
  return (...);
});

// ✅ Lazy loading — 頁面級元件
const UsersPage = lazy(() => import("@/features/users/components/UsersPage"));
```

**禁止**：

- 對所有元件都加 `React.memo`（過早優化）
- 在 render 裡建立 inline function 傳給已被 memo 的子元件（破壞 memo 效果）
- 大型 list 不分頁或不使用虛擬化（> 200 行考慮 `@tanstack/virtual`）

---

### 11. ESLint 規範 — 強制程式碼品質

以下規則必須通過（`lint` 不得有 error）：

```json
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn",
    "no-unused-vars": "error",
    "prefer-const": "error",
    "no-var": "error"
  }
}
```

**每次實作完執行**：

```bash
npm run lint
npm run type-check
```

---

### 12. 命名規範

| 類型         | 規範                            | 範例                              |
| ------------ | ------------------------------- | --------------------------------- |
| 元件         | `PascalCase`                    | `UserCard`, `UserForm`            |
| Hook         | `camelCase`，`use` 開頭         | `useUsers`, `useUserFilters`      |
| 函數         | `camelCase`，動詞開頭           | `handleSubmit`, `formatDate`      |
| 型別/介面    | `PascalCase`                    | `User`, `UserFilters`, `ApiError` |
| Enum         | `PascalCase`，成員 `PascalCase` | `UserRole.Admin`                  |
| 常數         | `UPPER_SNAKE_CASE`              | `USER_PAGE_SIZE`                  |
| 檔案（元件） | `PascalCase.tsx`                | `UserCard.tsx`                    |
| 檔案（其他） | `kebab-case.ts`                 | `use-users.ts`, `users.api.ts`    |
| CSS class    | `kebab-case`                    | `user-card`, `is-active`          |

---

### 13. 實作 Checklist

每次新增 feature 前，確認以下項目：

**型別與常數**

- [ ] `types/index.ts` 定義所有 DTO、Enum、UI 型別
- [ ] `constants/index.ts` 定義 label map、config 值
- [ ] 禁止 magic string / magic number 散落在元件

**API 層**

- [ ] `[feature].api.ts` 繼承 `HttpClient`
- [ ] `[feature].keys.ts` 定義 query key factory

**Hooks 層**

- [ ] 每個 hook 單一職責（資料 / UI state / 業務邏輯分開）
- [ ] Query hook 有 `enabled` guard（避免空 id 觸發請求）
- [ ] Mutation hook 在 `onSuccess` 做 `invalidateQueries`

**元件層**

- [ ] Page 元件只做資料獲取與組裝
- [ ] Presentational 元件只接受 props，不直接呼叫 API
- [ ] Props interface 定義在元件檔頂部，有完整型別
- [ ] Boolean props 使用 `is/has/can` 前綴
- [ ] Event handler props 使用 `on` 前綴

**表單**

- [ ] Zod schema 獨立定義在 `[Feature]Form.schema.ts`
- [ ] 使用 `zodResolver` 整合 React Hook Form
- [ ] `isSubmitting` 狀態傳入表單禁用送出按鈕

**錯誤處理**

- [ ] API 錯誤有 `onError` 處理（全局或 feature-level）
- [ ] 頁面有 `<ErrorBoundary>` 包覆
- [ ] 無 bare catch 吞掉 error

**效能**

- [ ] List 元件有 `key` prop（不用 index）
- [ ] 昂貴計算使用 `useMemo`
- [ ] 傳給已 memo 子元件的 handler 使用 `useCallback`

**品質**

- [ ] `npm run lint` 通過，無 error
- [ ] `npm run type-check` 通過，無 error
- [ ] 元件單一檔案不超過 200 行

---

### 14. Unit Test（可選）

開始實作前，**詢問使用者是否要一併撰寫 Unit Test**：

> 「是否需要為此功能撰寫 Unit Test？（y/n）」

若使用者選擇 **是**，在完成每個層次後，緊接著為該層次撰寫對應的測試：

**測試範圍與規範**：

```
Hooks      → 使用 renderHook + QueryClient wrapper，驗證資料狀態與 mutation 行為
元件        → 使用 @testing-library/react，驗證 render 輸出與使用者互動
純函數/工具  → 直接驗證輸入輸出，無需 mock
```

**檔案結構（測試檔放在被測試檔旁）**：

```
features/users/
├── hooks/
│   ├── use-users.ts
│   └── use-users.test.ts
├── components/
│   ├── UserCard.tsx
│   └── UserCard.test.tsx
└── utils/
    ├── format-user.ts
    └── format-user.test.ts
```

**測試範例**：

```typescript
// hooks/use-users.test.ts
import { renderHook, waitFor } from "@testing-library/react";
import { createWrapper } from "@/test/utils";
import { useUsers } from "./use-users";

describe("useUsers", () => {
  it("should return users list on success", async () => {
    const { result } = renderHook(() => useUsers(), {
      wrapper: createWrapper(),
    });
    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toHaveLength(2);
  });
});
```

```typescript
// components/UserCard.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { UserCard } from "./UserCard";

describe("UserCard", () => {
  it("renders user name and email", () => {
    render(<UserCard user={mockUser} onEdit={vi.fn()} />);
    expect(screen.getByText(mockUser.name)).toBeInTheDocument();
    expect(screen.getByText(mockUser.email)).toBeInTheDocument();
  });

  it("calls onEdit with user id when edit button clicked", async () => {
    const onEdit = vi.fn();
    render(<UserCard user={mockUser} onEdit={onEdit} />);
    await userEvent.click(screen.getByRole("button", { name: /編輯/ }));
    expect(onEdit).toHaveBeenCalledWith(mockUser.id);
  });
});
```

**Checklist（unit test 選項啟用時）**：

- [ ] 純函數 / 工具函數：正常路徑 + 邊界情況皆有覆蓋
- [ ] Hooks：以 `renderHook` 驗證 query / mutation 狀態變化
- [ ] 元件：驗證 render 輸出與主要使用者互動（click、input）
- [ ] 測試命名清楚描述情境：`it("should <behavior> when <condition>")`
- [ ] 執行 `npm run test` 全部通過後才完成

---

## 實作流程（程式碼層次順序）

> **模式一（單獨使用）**：依序執行步驟 0–8，完成後結束。
> **模式二（plan-step loop）**：在**第一個 plan-step** 執行步驟 0 詢問 Unit Test，之後所有 plan-step 套用該結論，不再重複詢問；完成當前層次後立即停止回報。

0. **詢問 Unit Test** → 確認使用者是否要一併撰寫 Unit Test（模式一：每次詢問；模式二：僅第一個 plan-step 詢問，後續 plan-step 略過此步驟）
1. **分析需求** → 識別需要哪些 API、型別、元件、hooks
2. **型別定義** → `features/[name]/types/index.ts` 先把 interface 定好
3. **常數定義** → `features/[name]/constants/index.ts` 定義 enum label、config
4. **API 層** → `[feature].api.ts` + `[feature].keys.ts`
5. **Hooks** → query hooks、mutation hooks、UI state hooks `→ 若選擇 Unit Test，同步撰寫 Hook 測試`
6. **元件** → Presentational 元件（純 UI）→ Form 元件 → Page 元件 `→ 若選擇 Unit Test，同步撰寫元件測試`
7. **Barrel export** → `index.ts` 公開外部需要的介面
8. **Lint / Type check** → 確保品質

每個層次完成後才進行下一層次，**不跳層、不混層**。

---

## 任務

**模式一（單獨使用）**：
請根據使用者描述的功能需求，嚴格依照上述規範實作完整 feature 模組。
開始前先詢問使用者是否需要撰寫 Unit Test。
若需求不清楚，先提問釐清再實作。

**模式二（plan-step loop）**：
依照當前 plan-step 的範圍，套用上述品質規範實作對應層次。
若為第一個 plan-step，先執行步驟 0 詢問 Unit Test，並將結果套用至後續所有 plan-step。
完成後回報並停止，**禁止自動繼續下一個 plan-step**。
