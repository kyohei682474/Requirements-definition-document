# Vision Todo v3 開発ガイド

このプロジェクトの開発規約を1ファイルにまとめたものです。

---

## 1. 基本方針：Feature-First アーキテクチャ

### コードを書ける場所

| ディレクトリ | 用途 |
|-------------|------|
| `src/features/*` | 機能専用コンポーネント・フック |
| `src/hooks/swr/*` | SWRデータ取得フック |
| `src/hooks/mutations/*` | ミューテーションフック |
| `src/lib/api/*` | APIクライアント |
| `src/types/models/*` | データベースモデル型 |

### 触ってはいけない場所

| ディレクトリ | 理由 |
|-------------|------|
| `src/components/*` | レビュアーが品質管理 |
| `src/hooks/*` (swr, mutations以外) | グローバルフック |
| `src/[feature-name]/*` | deprecated |
| `src/lib/types/*` | 使用禁止 |

---

## 2. Features ディレクトリ構造

```
features/[directory-name]/
├── [DirectoryName]Page.tsx     # Public（プレフィックス付き）
├── [DirectoryName]List.tsx     # Public
├── _components/                 # Private（プレフィックス不要）
│   ├── ListItem.tsx
│   └── ListHeader.tsx
└── hooks/
    └── useSelection.ts
```

### 命名規則

- **Public** (トップレベル): `[DirectoryName]X.tsx` → `UsersPage.tsx`
- **Private** (`_components/`内): 短い名前 → `ListItem.tsx`

---

## 3. コンポーネントのベストプラクティス

### State は最小スコープに配置

```tsx
// ✅ Good: State が使用される場所に配置
function UsersPage() {
  return (
    <>
      <UsersList />    {/* selectedId はここに */}
      <SortControls /> {/* sortOrder はここに */}
    </>
  );
}

// ❌ Bad: トップレベルで全State管理
function UsersPage() {
  const [selectedId, setSelectedId] = useState(null);
  const [sortOrder, setSortOrder] = useState("asc");
  return (
    <>
      <UsersList selectedId={selectedId} onSelect={setSelectedId} />
      <SortControls order={sortOrder} onChange={setSortOrder} />
    </>
  );
}
```

### ロジックは子コンポーネントに押し下げる

```tsx
// ✅ Good: 子が自分のロジックを持つ
function UserCard({ user }: { user: User }) {
  const { deleteUser } = useUsersMutation();

  const handleDelete = async () => {
    await deleteUser(user.id);
  };

  return <button onClick={handleDelete}>削除</button>;
}
```

### コンポーネント構造

- メインコンポーネント(export)を**先頭**に
- Loading/Error/Emptyは**ファイル末尾**に

```tsx
export function UsersList() {
  const { data, isLoading, error } = useUsersSWR();

  if (isLoading) return <Loading />;
  if (error) return <Error />;
  if (!data?.length) return <Empty />;

  return <div>{/* メイン */}</div>;
}

// ファイル末尾
function Loading() { return <div>読み込み中...</div>; }
function Error() { return <div>エラー</div>; }
function Empty() { return <div>データなし</div>; }
```

### Dialog/Modal: shadcn Trigger + Content

```tsx
// ✅ Good: インラインで配置
<Dialog>
  <DialogTrigger asChild>
    <Button>追加</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogTitle>ユーザー追加</DialogTitle>
  </DialogContent>
</Dialog>

// ❌ Bad: state で分離管理
const [isOpen, setIsOpen] = useState(false);
<Button onClick={() => setIsOpen(true)}>追加</Button>
<AddUserModal isOpen={isOpen} onClose={() => setIsOpen(false)} />
```

---

## 4. SWR フック

### 配置場所とファイル名

`src/hooks/swr/useXSWR.ts`

| パターン | 例 |
|---------|-----|
| 単数形 | `useUserSWR.ts` - 1つのリソース |
| 複数形 | `useUsersSWR.ts` - リスト |

### 基本パターン

```tsx
// src/hooks/swr/useUserSWR.ts
export function useUserSWR(
  userId: string | null | undefined,
  config?: SWRConfiguration,
  disabled?: boolean,
) {
  const key = userId && !disabled ? ["users", userId] : null;

  return useSWR(key, () => usersApi.get(userId!), {
    revalidateOnFocus: true,
    revalidateOnReconnect: true,
    ...config,
  });
}
```

### データが必要な場所で呼ぶ

```tsx
// ✅ Good: 必要な場所で取得
function UsersList() {
  const organizationId = useOrganizationContext();
  const { data: users } = useUsersSWR(organizationId);
  return (/* ... */);
}

// ❌ Bad: 親で取得して渡す
function UsersPage() {
  const { data: users } = useUsersSWR(organizationId);
  return <UsersList users={users} />;
}
```

---

## 5. API クライアント

### 配置場所とファイル名

`src/lib/api/[resource].ts` (複数形)

### 基本パターン

```tsx
// src/lib/api/users.ts
import { apiClient } from "@/lib/api/client";
import type { User } from "@/types/models/user";

// usersApi.create
type CreateUserRequest = Omit<User, "id" | "createdAt" | "updatedAt">;

// usersApi.update
type UpdateUserRequest = Partial<Pick<User, "name" | "email">>;

export const usersApi = {
  list: async () => apiClient.get<User[]>("/users"),

  get: async (userId: string) => apiClient.get<User>(`/users/${userId}`),

  create: async (data: CreateUserRequest) =>
    apiClient.post<{ user: User }>("/users", data),

  update: async (userId: string, data: UpdateUserRequest) =>
    apiClient.patch<{ user: User }>(`/users/${userId}`, data),

  delete: async (userId: string) => apiClient.delete(`/users/${userId}`),
};
```

### ポイント

- 型は同ファイルに定義
- コメントで対応メソッドを明示 (`// usersApi.create`)
- 型は `apiClient.get<Type>()` で指定（戻り値型アノテーション不要）

---

## 6. Mutation フック

### 配置場所とファイル名

`src/hooks/mutations/use[DataName]Mutation.ts`

### 基本パターン

```tsx
// src/hooks/mutations/useUsersMutation.ts
import { mutate } from "swr";
import { usersApi } from "@/lib/api/users";
import type { User } from "@/types/models/user";

export function useUsersMutation() {
  const updateUser = async (userId: string, data: Partial<User>) => {
    const updatedUser = await usersApi.update(userId, data);

    // 関連キャッシュを更新
    mutate(["users", userId]);
    mutate(["users"]);

    return updatedUser;
  };

  const deleteUser = async (userId: string) => {
    await usersApi.delete(userId);
    mutate(["users", userId], undefined, { revalidate: false });
    mutate(["users"]);
  };

  return { updateUser, deleteUser };
}
```

### 楽観的更新

```tsx
const updateUser = async (userId: string, data: Partial<User>) => {
  // 1. 先にUIを更新
  mutate(
    ["users", userId],
    (current: User | undefined) => current ? { ...current, ...data } : current,
    { revalidate: false },
  );

  try {
    // 2. API呼び出し
    const updated = await usersApi.update(userId, data);
    mutate(["users", userId], updated, { revalidate: false });
    return updated;
  } catch (error) {
    // 3. 失敗したら元に戻す
    mutate(["users", userId]);
    throw error;
  }
};
```

---

## 7. 型定義

### 配置場所

| 場所 | 内容 |
|------|------|
| `src/types/models/*` | DBモデルの基本型 |
| `src/lib/api/*` | Request/Response型（基本型から派生） |

### ルール

- **常に `type` を使用** (`interface` は使わない)
- ステータス・ロールには `enum` を使用

### 基本型の例

```tsx
// src/types/models/objective.ts
export enum ObjectiveStatus {
  ACTIVE = "active",
  COMPLETED = "completed",
  ARCHIVED = "archived",
}

export type Objective = {
  id: string;
  status: ObjectiveStatus;
  title: string;
  createdAt: string;
  updatedAt: string;
};
```

### 派生型のパターン

```tsx
// 作成: 自動生成フィールドを除外
type CreateRequest = Omit<User, "id" | "createdAt" | "updatedAt">;

// 更新: 変更可能フィールドのみ選択 + オプショナル
type UpdateRequest = Partial<Pick<User, "name" | "email">>;

// レスポンス: 追加フィールドを含む
type DetailResponse = User & { postCount: number };
```

---

## 8. Context

### 使用方法

```tsx
function MyComponent() {
  const organizationId = useOrganizationContext();
  const organizationMemberId = useOrganizationMemberContext();
}
```

### ルール

- ✅ `useOrganizationContext()` / `useOrganizationMemberContext()` を使用
- ❌ IDをpropsで渡さない（必要なコンポーネントでContextを使う）
- ❌ `useOrganizationId()` は使わない (deprecated)
- ❌ IDだけ欲しいなら `useCurrentOrganizationMember()` は使わない

---

## 9. フォーム (React Hook Form + Zod)

### スキーマ定義

```tsx
// src/features/users/schemas/userSchema.ts
import { z } from "zod";

export const updateUserSchema = z.object({
  name: z.string().min(1, "名前は必須です"),
  email: z.string().email("有効なメールアドレスを入力してください"),
});

export type UpdateUserFormData = z.infer<typeof updateUserSchema>;
```

### フォームコンポーネント

```tsx
function UserEditForm({ user }: { user: User }) {
  const { updateUser } = useUsersMutation();
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<UpdateUserFormData>({
    resolver: zodResolver(updateUserSchema),
    defaultValues: { name: user.name, email: user.email },
  });

  const onSubmit = async (data: UpdateUserFormData) => {
    try {
      await updateUser(user.id, data);
      toast.success("更新しました");
    } catch {
      toast.error("更新に失敗しました");
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name")} />
      {errors.name && <span>{errors.name.message}</span>}
      <button disabled={isSubmitting}>
        {isSubmitting ? "保存中..." : "保存"}
      </button>
    </form>
  );
}
```

---

## クイックリファレンス

| やりたいこと | 場所 | 命名 |
|-------------|------|------|
| 新機能のコンポーネント | `src/features/[name]/` | `[Name]Page.tsx` |
| プライベートコンポーネント | `src/features/[name]/_components/` | `ListItem.tsx` |
| データ取得 | `src/hooks/swr/` | `useUserSWR.ts` |
| データ変更 | `src/hooks/mutations/` | `useUsersMutation.ts` |
| API呼び出し | `src/lib/api/` | `users.ts` → `usersApi` |
| 基本型 | `src/types/models/` | `user.ts` |
| フォームスキーマ | `src/features/[name]/schemas/` | `userSchema.ts` |
