# TypeScript - ユニオン型とインターセクション型を使いこなす

## はじめに

TypeScriptの型システムの中でも、ユニオン型（`|`）とインターセクション型（`&`）は非常によく使う重要な機能です。

ユニオン型は「AまたはB」、インターセクション型は「AかつB」を表現します。

## ユニオン型（Union Types）

### 基本的な使い方

```typescript
// string または number を受け入れる
type StringOrNumber = string | number;

function printId(id: StringOrNumber) {
  console.log(`ID: ${id}`);
}

printId(101);      // OK
printId("abc");    // OK
// printId(true);  // エラー！
```

### リテラル型のユニオン

特定の値のみを許可する型を定義できます。

```typescript
type Direction = "north" | "south" | "east" | "west";
type HttpStatus = 200 | 301 | 400 | 404 | 500;
type Size = "small" | "medium" | "large";

function move(direction: Direction) {
  console.log(`Moving ${direction}`);
}

move("north"); // OK
// move("up"); // エラー！
```

### ユニオン型の絞り込み（ナローイング）

ユニオン型の変数を使うには、どの型なのかを絞り込む必要があります。

```typescript
function formatValue(value: string | number): string {
  // typeof による絞り込み
  if (typeof value === "string") {
    return value.toUpperCase(); // string のメソッドが使える
  }
  return value.toFixed(2); // number のメソッドが使える
}
```

### Discriminated Union（判別可能なユニオン）

実務で最も重要なパターンの一つです。共通のプロパティ（判別子）を持たせることで、型を安全に絞り込めます。

```typescript
type Success = {
  type: "success";
  data: string;
};

type Loading = {
  type: "loading";
};

type ErrorState = {
  type: "error";
  message: string;
  code: number;
};

type RequestState = Success | Loading | ErrorState;

function handleState(state: RequestState) {
  switch (state.type) {
    case "success":
      console.log(state.data);    // data にアクセス可能
      break;
    case "loading":
      console.log("読み込み中...");
      break;
    case "error":
      console.log(state.message); // message にアクセス可能
      console.log(state.code);    // code にアクセス可能
      break;
  }
}
```

### 網羅性チェック

`never` 型を使って、すべてのケースを処理したか確認できます。

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function handleState(state: RequestState): string {
  switch (state.type) {
    case "success":
      return state.data;
    case "loading":
      return "読み込み中...";
    case "error":
      return state.message;
    default:
      return assertNever(state); // すべてのケースを処理していないとエラー
  }
}
```

新しいケースが追加されたとき、`default` の `assertNever` がコンパイルエラーになるため、処理の追加忘れを防げます。

## インターセクション型（Intersection Types）

### 基本的な使い方

複数の型を「合成」します。

```typescript
type HasName = {
  name: string;
};

type HasAge = {
  age: number;
};

type HasEmail = {
  email: string;
};

type Person = HasName & HasAge & HasEmail;

const person: Person = {
  name: "太郎",
  age: 25,
  email: "taro@example.com",
};
```

### ミックスインパターン

既存の型に機能を追加するパターンです。

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
  isDeleted: boolean;
};

type BaseEntity = {
  id: string;
};

// すべてを合成
type User = BaseEntity &
  Timestamped &
  SoftDeletable & {
    name: string;
    email: string;
  };
```

### 関数型のインターセクション

```typescript
type Logger = {
  log(message: string): void;
};

type ErrorReporter = {
  reportError(error: Error): void;
};

type Debugger = Logger & ErrorReporter;

const debugTool: Debugger = {
  log(message) {
    console.log(message);
  },
  reportError(error) {
    console.error(error);
  },
};
```

## ユニオン型とインターセクション型の組み合わせ

### 複雑な型の表現

```typescript
type AdminPermission = "manage_users" | "manage_settings" | "view_logs";
type EditorPermission = "edit_content" | "publish_content";
type ViewerPermission = "view_content";

type BaseUser = {
  id: string;
  name: string;
  email: string;
};

type Admin = BaseUser & {
  role: "admin";
  permissions: AdminPermission[];
};

type Editor = BaseUser & {
  role: "editor";
  permissions: EditorPermission[];
};

type Viewer = BaseUser & {
  role: "viewer";
  permissions: ViewerPermission[];
};

type AppUser = Admin | Editor | Viewer;

function getPermissions(user: AppUser): string[] {
  switch (user.role) {
    case "admin":
      return user.permissions; // AdminPermission[]
    case "editor":
      return user.permissions; // EditorPermission[]
    case "viewer":
      return user.permissions; // ViewerPermission[]
  }
}
```

### 条件付き型との組み合わせ

```typescript
type ApiEndpoint =
  | { method: "GET"; url: string; params?: Record<string, string> }
  | { method: "POST"; url: string; body: unknown }
  | { method: "PUT"; url: string; body: unknown }
  | { method: "DELETE"; url: string };

async function callApi(endpoint: ApiEndpoint): Promise<unknown> {
  const { method, url } = endpoint;

  switch (method) {
    case "GET":
      const queryString = endpoint.params
        ? "?" + new URLSearchParams(endpoint.params).toString()
        : "";
      return fetch(url + queryString);
    case "POST":
    case "PUT":
      return fetch(url, {
        method,
        body: JSON.stringify(endpoint.body),
      });
    case "DELETE":
      return fetch(url, { method });
  }
}
```

## 注意点

### プリミティブ型のインターセクション

プリミティブ型同士のインターセクションは `never` になります。

```typescript
type Impossible = string & number; // never
```

### ユニオン型の共通プロパティ

ユニオン型では、すべてのメンバーに共通するプロパティのみアクセスできます。

```typescript
type Cat = { name: string; purr(): void };
type Dog = { name: string; bark(): void };

type Pet = Cat | Dog;

function greetPet(pet: Pet) {
  console.log(pet.name); // OK: 両方に name がある
  // pet.purr();          // エラー！Dog には purr がない
  // pet.bark();          // エラー！Cat には bark がない
}
```

## まとめ

- ユニオン型（`|`）は「AまたはB」を表現する
- インターセクション型（`&`）は「AかつB」を表現する
- Discriminated Unionは実務で最も重要なパターン
- `never` 型で網羅性チェックを行うとバグを防げる
- 両者を組み合わせることで、複雑なドメインモデルも型安全に表現できる

次回は「型ガードとナローイング」について詳しく解説します。
