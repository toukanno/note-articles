# TypeScript ユーティリティ型 完全ガイド

## はじめに

TypeScriptには、型変換を簡単にするための「ユーティリティ型（Utility Types）」が多数組み込まれています。これらを使いこなすことで、既存の型から新しい型を効率的に導出できます。

## オブジェクト操作系

### Partial<T>

すべてのプロパティをオプショナルにします。

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string }

// ユースケース：更新時に一部のフィールドだけ指定
function updateUser(id: number, updates: Partial<User>) {
  // updates は一部のフィールドだけでOK
}

updateUser(1, { name: "新しい名前" }); // OK
```

### Required<T>

すべてのプロパティを必須にします。`Partial` の逆です。

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

type RequiredConfig = Required<Config>;
// { host: string; port: number; debug: boolean }
```

### Readonly<T>

すべてのプロパティを読み取り専用にします。

```typescript
type ReadonlyUser = Readonly<User>;

const user: ReadonlyUser = { id: 1, name: "太郎", email: "taro@example.com" };
// user.name = "花子"; // エラー！readonly
```

### Pick<T, K>

指定したプロパティだけを抽出します。

```typescript
type UserSummary = Pick<User, "id" | "name">;
// { id: number; name: string }
```

### Omit<T, K>

指定したプロパティを除外します。`Pick` の逆です。

```typescript
type UserWithoutEmail = Omit<User, "email">;
// { id: number; name: string }

// ユースケース：作成時にIDを除外
type CreateUserInput = Omit<User, "id">;
// { name: string; email: string }
```

### Record<K, V>

キーの型と値の型からオブジェクト型を生成します。

```typescript
type UserRoles = Record<string, string[]>;

const roles: UserRoles = {
  admin: ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};

// リテラルユニオンをキーに
type PageViews = Record<"home" | "about" | "contact", number>;
```

## ユニオン操作系

### Exclude<T, U>

ユニオン型から特定の型を除外します。

```typescript
type Status = "active" | "inactive" | "pending" | "deleted";
type ActiveStatus = Exclude<Status, "deleted">;
// "active" | "inactive" | "pending"

type NonNullString = Exclude<string | null | undefined, null | undefined>;
// string
```

### Extract<T, U>

ユニオン型から特定の型だけを抽出します。

```typescript
type StringOrNumber = Extract<string | number | boolean, string | number>;
// string | number
```

### NonNullable<T>

`null` と `undefined` を除外します。

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;
// string
```

## 関数操作系

### ReturnType<T>

関数の戻り値の型を取得します。

```typescript
function createUser() {
  return { id: 1, name: "太郎", createdAt: new Date() };
}

type User = ReturnType<typeof createUser>;
// { id: number; name: string; createdAt: Date }
```

### Parameters<T>

関数の引数の型をタプルとして取得します。

```typescript
function search(query: string, page: number, limit: number) {
  // ...
}

type SearchParams = Parameters<typeof search>;
// [query: string, page: number, limit: number]

// 個別の引数を取得
type FirstParam = Parameters<typeof search>[0]; // string
```

### ConstructorParameters<T>

コンストラクタの引数の型を取得します。

```typescript
class UserService {
  constructor(
    private apiUrl: string,
    private timeout: number
  ) {}
}

type ServiceParams = ConstructorParameters<typeof UserService>;
// [apiUrl: string, timeout: number]
```

### InstanceType<T>

クラスのインスタンス型を取得します。

```typescript
type UserServiceInstance = InstanceType<typeof UserService>;
// UserService
```

## 文字列操作系

```typescript
type A = Uppercase<"hello">;     // "HELLO"
type B = Lowercase<"HELLO">;     // "hello"
type C = Capitalize<"hello">;    // "Hello"
type D = Uncapitalize<"Hello">;  // "hello"
```

## Promise系

### Awaited<T>

`Promise` をアンラップして中身の型を取得します。

```typescript
type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number
type C = Awaited<string | Promise<number>>;  // string | number
```

## 実践的な組み合わせパターン

### 作成・更新・レスポンスの型を一元管理

```typescript
interface UserEntity {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// 作成時：id と日付は自動生成
type CreateUser = Omit<UserEntity, "id" | "createdAt" | "updatedAt">;

// 更新時：一部のフィールドのみ
type UpdateUser = Partial<Omit<UserEntity, "id" | "createdAt" | "updatedAt">>;

// 一覧表示用：必要なフィールドだけ
type UserListItem = Pick<UserEntity, "id" | "name" | "role">;

// レスポンス：読み取り専用
type UserResponse = Readonly<UserEntity>;
```

### フォームの状態管理

```typescript
interface FormData {
  username: string;
  email: string;
  password: string;
}

// フォームの入力状態
type FormState = {
  values: FormData;
  errors: Partial<Record<keyof FormData, string>>;
  touched: Partial<Record<keyof FormData, boolean>>;
  isSubmitting: boolean;
};
```

### API エンドポイントの型定義

```typescript
interface ApiEndpoints {
  "/users": {
    GET: { response: User[]; query: { page: number; limit: number } };
    POST: { response: User; body: CreateUser };
  };
  "/users/:id": {
    GET: { response: User };
    PUT: { response: User; body: UpdateUser };
    DELETE: { response: void };
  };
}

type GetResponse<
  Path extends keyof ApiEndpoints,
  Method extends keyof ApiEndpoints[Path]
> = ApiEndpoints[Path][Method] extends { response: infer R } ? R : never;

type UsersResponse = GetResponse<"/users", "GET">; // User[]
```

### ReadonlyDeep

```typescript
type ReadonlyDeep<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : ReadonlyDeep<T[K]>
    : T[K];
};
```

### PartialDeep

```typescript
type PartialDeep<T> = {
  [K in keyof T]?: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : PartialDeep<T[K]>
    : T[K];
};
```

## まとめ

| ユーティリティ型 | 説明 |
|---|---|
| `Partial<T>` | すべてオプショナル |
| `Required<T>` | すべて必須 |
| `Readonly<T>` | すべて読み取り専用 |
| `Pick<T, K>` | 指定プロパティを抽出 |
| `Omit<T, K>` | 指定プロパティを除外 |
| `Record<K, V>` | キーと値の型からオブジェクト生成 |
| `Exclude<T, U>` | ユニオンから除外 |
| `Extract<T, U>` | ユニオンから抽出 |
| `NonNullable<T>` | null/undefined を除外 |
| `ReturnType<T>` | 関数の戻り値型 |
| `Parameters<T>` | 関数の引数型 |
| `Awaited<T>` | Promise をアンラップ |

次回は「enumの使い方と注意点」について詳しく解説します。
