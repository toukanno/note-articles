# TypeScript ジェネリクス入門 - 柔軟で型安全なコードを書く

## はじめに

ジェネリクス（Generics）は、TypeScriptの中でも特に強力な機能です。型をパラメータ化することで、さまざまな型に対応しながらも型安全性を保つコードが書けます。

「どんな型でも受け入れられるが、一度決まったら一貫性を保つ」——それがジェネリクスの本質です。

## なぜジェネリクスが必要か

### any を使う場合の問題

```typescript
function getFirst(arr: any[]): any {
  return arr[0];
}

const result = getFirst([1, 2, 3]);
// result は any 型 → 型情報が失われる
result.toUpperCase(); // コンパイルエラーにならない！（実行時エラー）
```

### ジェネリクスで解決

```typescript
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

const result = getFirst([1, 2, 3]);
// result は number 型 → 型情報が保持される
// result.toUpperCase(); // コンパイルエラー！number にはない
```

## 基本構文

### ジェネリック関数

```typescript
// 基本形
function identity<T>(value: T): T {
  return value;
}

// 使用時に型を明示
const str = identity<string>("hello");

// 型推論に任せる（推奨）
const num = identity(42); // T は number と推論
```

### アロー関数での書き方

```typescript
const identity = <T>(value: T): T => {
  return value;
};

// TSXファイルでは <T,> と書く（JSXタグとの区別のため）
const identity2 = <T,>(value: T): T => value;
```

### 複数の型パラメータ

```typescript
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const result = pair("hello", 42); // [string, number]
```

## ジェネリック型

### ジェネリックインターフェース

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// 使用例
interface User {
  id: number;
  name: string;
}

const response: ApiResponse<User> = {
  data: { id: 1, name: "太郎" },
  status: 200,
  message: "OK",
};

const listResponse: ApiResponse<User[]> = {
  data: [{ id: 1, name: "太郎" }],
  status: 200,
  message: "OK",
};
```

### ジェネリック型エイリアス

```typescript
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
```

### ジェネリッククラス

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
const top = numberStack.pop(); // number | undefined
```

## 型制約（extends）

ジェネリクスに制約を加えることで、特定のプロパティやメソッドを持つ型に限定できます。

### 基本的な制約

```typescript
// length プロパティを持つ型に限定
function logLength<T extends { length: number }>(value: T): T {
  console.log(`長さ: ${value.length}`);
  return value;
}

logLength("hello");      // OK: string は length を持つ
logLength([1, 2, 3]);    // OK: 配列は length を持つ
// logLength(42);         // エラー！number は length を持たない
```

### keyof を使った制約

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "太郎", age: 25, email: "taro@example.com" };

const name = getProperty(user, "name");   // string
const age = getProperty(user, "age");     // number
// getProperty(user, "phone");            // エラー！"phone" は User のキーにない
```

### インターフェースによる制約

```typescript
interface Identifiable {
  id: string | number;
}

function findById<T extends Identifiable>(items: T[], id: T["id"]): T | undefined {
  return items.find((item) => item.id === id);
}

interface Product {
  id: number;
  name: string;
  price: number;
}

const products: Product[] = [
  { id: 1, name: "りんご", price: 150 },
  { id: 2, name: "みかん", price: 100 },
];

const found = findById(products, 1); // Product | undefined
```

## デフォルト型パラメータ

型パラメータにデフォルト値を設定できます。

```typescript
interface PaginatedResponse<T, M = { total: number; page: number }> {
  data: T[];
  meta: M;
}

// デフォルトのメタデータ型を使用
const response: PaginatedResponse<User> = {
  data: [{ id: 1, name: "太郎" }],
  meta: { total: 100, page: 1 },
};

// カスタムのメタデータ型を指定
const response2: PaginatedResponse<User, { cursor: string }> = {
  data: [{ id: 1, name: "太郎" }],
  meta: { cursor: "abc123" },
};
```

## 実践的な使用例

### 型安全なイベントエミッター

```typescript
type EventMap = {
  login: { userId: string; timestamp: number };
  logout: { userId: string };
  error: { message: string; code: number };
};

class TypedEventEmitter<T extends Record<string, unknown>> {
  private listeners: Partial<{
    [K in keyof T]: Array<(data: T[K]) => void>;
  }> = {};

  on<K extends keyof T>(event: K, listener: (data: T[K]) => void): void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
  }

  emit<K extends keyof T>(event: K, data: T[K]): void {
    this.listeners[event]?.forEach((listener) => listener(data));
  }
}

const emitter = new TypedEventEmitter<EventMap>();

emitter.on("login", (data) => {
  console.log(data.userId);    // string と推論
  console.log(data.timestamp); // number と推論
});

emitter.emit("login", { userId: "123", timestamp: Date.now() }); // OK
// emitter.emit("login", { userId: "123" }); // エラー！timestamp が必要
```

### 型安全なHTTPクライアント

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

async function fetchApi<T>(
  url: string,
  method: HttpMethod = "GET",
  body?: unknown
): Promise<ApiResponse<T>> {
  const response = await fetch(url, {
    method,
    headers: { "Content-Type": "application/json" },
    body: body ? JSON.stringify(body) : undefined,
  });

  return response.json() as Promise<ApiResponse<T>>;
}

// 使用例
interface User {
  id: number;
  name: string;
}

const users = await fetchApi<User[]>("/api/users");
// users.data は User[] 型
```

## よくある間違い

### 不要なジェネリクス

```typescript
// 悪い例：ジェネリクスが不要
function greet<T extends string>(name: T): string {
  return `Hello, ${name}`;
}

// 良い例
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

ジェネリクスは「型を関連付ける」ために使います。引数と戻り値の間に関連がない場合、ジェネリクスは不要です。

## まとめ

- ジェネリクスは型をパラメータ化し、柔軟かつ型安全なコードを実現する
- `extends` で型制約を加えられる
- `keyof` と組み合わせるとオブジェクトの型安全な操作が可能
- デフォルト型パラメータで使い勝手を向上できる
- `any` の代わりにジェネリクスを使うことで、型情報を保持しよう

次回は「ユニオン型とインターセクション型」について詳しく解説します。
