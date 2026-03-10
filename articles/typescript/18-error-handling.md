# TypeScript - 型安全なエラーハンドリング

## はじめに

JavaScriptのエラーハンドリングはtry/catchが基本ですが、catchされるエラーは `unknown` 型であり、型安全性に課題があります。

この記事では、TypeScriptで型安全にエラーを扱うための様々なパターンを紹介します。

## try/catch の課題

### catch の error は unknown

```typescript
try {
  const data = JSON.parse(input);
} catch (error) {
  // error は unknown 型
  // error.message; // エラー！unknown にはプロパティがない

  // 型ガードが必要
  if (error instanceof Error) {
    console.log(error.message);
  }
}
```

### throw は何でも投げられる

```typescript
throw new Error("エラーメッセージ");
throw "文字列も投げられる";
throw 42;
throw { code: "E001", message: "エラー" };
throw null;
```

TypeScriptは `throw` の型を制限できません。

## カスタムエラークラス

### 基本パターン

```typescript
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500
  ) {
    super(message);
    this.name = "AppError";
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, "NOT_FOUND", 404);
    this.name = "NotFoundError";
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string[]>
  ) {
    super(message, "VALIDATION_ERROR", 400);
    this.name = "ValidationError";
  }
}

class AuthenticationError extends AppError {
  constructor(message: string = "認証が必要です") {
    super(message, "UNAUTHORIZED", 401);
    this.name = "AuthenticationError";
  }
}
```

### 使用例

```typescript
function handleError(error: unknown) {
  if (error instanceof ValidationError) {
    console.log("バリデーションエラー:", error.fields);
  } else if (error instanceof NotFoundError) {
    console.log("リソースが見つかりません:", error.message);
  } else if (error instanceof AuthenticationError) {
    console.log("認証エラー:", error.message);
  } else if (error instanceof AppError) {
    console.log(`アプリエラー [${error.code}]:`, error.message);
  } else if (error instanceof Error) {
    console.log("予期しないエラー:", error.message);
  } else {
    console.log("不明なエラー:", error);
  }
}
```

## Result型パターン

関数型プログラミングの影響を受けたパターンで、エラーを戻り値として表現します。

### 基本的なResult型

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

// ヘルパー関数
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}
```

### 使用例

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

type UserError =
  | { type: "NOT_FOUND"; userId: string }
  | { type: "VALIDATION"; fields: string[] }
  | { type: "NETWORK"; message: string };

function findUser(id: string): Result<User, UserError> {
  if (!id) {
    return err({ type: "VALIDATION", fields: ["id"] });
  }

  const user = database.get(id);
  if (!user) {
    return err({ type: "NOT_FOUND", userId: id });
  }

  return ok(user);
}

// 使用側
const result = findUser("123");

if (result.ok) {
  console.log(result.value.name); // User 型にアクセス
} else {
  switch (result.error.type) {
    case "NOT_FOUND":
      console.log(`ユーザー ${result.error.userId} が見つかりません`);
      break;
    case "VALIDATION":
      console.log(`入力エラー: ${result.error.fields.join(", ")}`);
      break;
    case "NETWORK":
      console.log(`通信エラー: ${result.error.message}`);
      break;
  }
}
```

### 非同期版Result

```typescript
type AsyncResult<T, E = Error> = Promise<Result<T, E>>;

async function fetchUser(id: string): AsyncResult<User, UserError> {
  try {
    const response = await fetch(`/api/users/${id}`);

    if (response.status === 404) {
      return err({ type: "NOT_FOUND", userId: id });
    }

    if (!response.ok) {
      return err({ type: "NETWORK", message: `HTTP ${response.status}` });
    }

    const user = await response.json();
    return ok(user as User);
  } catch (e) {
    return err({
      type: "NETWORK",
      message: e instanceof Error ? e.message : "Unknown error",
    });
  }
}
```

### Result のチェーン

```typescript
function map<T, U, E>(result: Result<T, E>, fn: (value: T) => U): Result<U, E> {
  if (result.ok) {
    return ok(fn(result.value));
  }
  return result;
}

function flatMap<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => Result<U, E>
): Result<U, E> {
  if (result.ok) {
    return fn(result.value);
  }
  return result;
}

// 使用例
const result = findUser("123");
const nameResult = map(result, (user) => user.name);
// Result<string, UserError>
```

## エラー境界パターン

### try/catch のラッパー

```typescript
function tryCatch<T>(fn: () => T): Result<T> {
  try {
    return ok(fn());
  } catch (error) {
    return err(error instanceof Error ? error : new Error(String(error)));
  }
}

async function tryCatchAsync<T>(fn: () => Promise<T>): AsyncResult<T> {
  try {
    const value = await fn();
    return ok(value);
  } catch (error) {
    return err(error instanceof Error ? error : new Error(String(error)));
  }
}

// 使用例
const result = tryCatch(() => JSON.parse(jsonString));
if (result.ok) {
  console.log(result.value);
} else {
  console.log("パースエラー:", result.error.message);
}
```

## Zodを使ったバリデーション

外部ライブラリのZodは、TypeScriptと相性の良いバリデーションライブラリです。

```typescript
import { z } from "zod";

// スキーマ定義
const UserSchema = z.object({
  name: z.string().min(1, "名前は必須です"),
  email: z.string().email("有効なメールアドレスを入力してください"),
  age: z.number().min(0).max(150),
});

// スキーマから型を導出
type User = z.infer<typeof UserSchema>;

// バリデーション
function validateUser(input: unknown): Result<User, z.ZodError> {
  const result = UserSchema.safeParse(input);

  if (result.success) {
    return ok(result.data);
  }

  return err(result.error);
}

// 使用例
const result = validateUser({ name: "", email: "invalid", age: -1 });

if (!result.ok) {
  const errors = result.error.errors.map((e) => ({
    field: e.path.join("."),
    message: e.message,
  }));
  console.log(errors);
  // [
  //   { field: "name", message: "名前は必須です" },
  //   { field: "email", message: "有効なメールアドレスを入力してください" },
  //   { field: "age", message: "..." }
  // ]
}
```

## アサーション関数

条件を満たさない場合に例外を投げる関数です。

```typescript
function assertDefined<T>(
  value: T | null | undefined,
  message?: string
): asserts value is T {
  if (value === null || value === undefined) {
    throw new AppError(
      message ?? "Unexpected null or undefined",
      "ASSERTION_ERROR"
    );
  }
}

function assertEqual<T>(
  actual: T,
  expected: T,
  message?: string
): void {
  if (actual !== expected) {
    throw new AppError(
      message ?? `Expected ${expected}, got ${actual}`,
      "ASSERTION_ERROR"
    );
  }
}

// 使用例
function processUser(userId: string) {
  const user = findUserById(userId);
  assertDefined(user, `User ${userId} not found`);

  // この行以降、user は null/undefined ではないことが保証される
  return user.name;
}
```

## 実践的なエラーハンドリング戦略

### レイヤーごとの責務

```typescript
// 1. ドメイン層：ビジネスルールのエラー
type DomainError =
  | { type: "INSUFFICIENT_BALANCE"; required: number; available: number }
  | { type: "ACCOUNT_LOCKED"; reason: string };

// 2. アプリケーション層：ユースケースのエラー
type ApplicationError =
  | DomainError
  | { type: "USER_NOT_FOUND"; userId: string }
  | { type: "UNAUTHORIZED" };

// 3. プレゼンテーション層：ユーザー向けのエラー変換
function toUserMessage(error: ApplicationError): string {
  switch (error.type) {
    case "INSUFFICIENT_BALANCE":
      return `残高が不足しています（必要: ${error.required}円、残高: ${error.available}円）`;
    case "ACCOUNT_LOCKED":
      return `アカウントがロックされています: ${error.reason}`;
    case "USER_NOT_FOUND":
      return "ユーザーが見つかりません";
    case "UNAUTHORIZED":
      return "ログインが必要です";
  }
}
```

## まとめ

- catch の error は `unknown` 型なので、型ガードが必要
- カスタムエラークラスで構造化されたエラーを定義できる
- Result型パターンでエラーを戻り値として型安全に扱える
- アサーション関数で実行時チェックと型ナローイングを両立できる
- Zodなどのバリデーションライブラリと組み合わせると効果的
- レイヤーごとにエラーの責務を分離すると保守性が向上する

次回は「TypeScriptのデザインパターン」について詳しく解説します。
