# TypeScript - 非同期処理の型付け

## はじめに

JavaScriptの非同期処理（Promise、async/await）はWebアプリケーション開発で欠かせません。TypeScriptでは、これらの非同期処理にも型をつけることで、より安全なコードが書けます。

## Promiseの型

### 基本的な型注釈

```typescript
// Promise<T> — T は解決後の値の型
const promise: Promise<string> = new Promise((resolve) => {
  resolve("完了");
});

// async 関数は自動的に Promise を返す
async function fetchUser(): Promise<User> {
  const response = await fetch("/api/user");
  return response.json();
}
```

### Promise の作成

```typescript
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function fetchWithTimeout<T>(
  promise: Promise<T>,
  ms: number
): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error("Timeout")), ms)
    ),
  ]);
}
```

## async/await の型

### 基本パターン

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

async function getUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  return response.json() as Promise<User>;
}

// 使用側
async function main() {
  const user = await getUser(1); // user は User 型
  console.log(user.name);
}
```

### エラーハンドリング

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safeAsync<T>(
  fn: () => Promise<T>
): Promise<Result<T>> {
  try {
    const value = await fn();
    return { ok: true, value };
  } catch (error) {
    return { ok: false, error: error instanceof Error ? error : new Error(String(error)) };
  }
}

// 使用例
async function main() {
  const result = await safeAsync(() => getUser(1));

  if (result.ok) {
    console.log(result.value.name); // User 型にアクセス
  } else {
    console.error(result.error.message); // Error 型にアクセス
  }
}
```

## Promise の並行処理

### Promise.all

すべてのPromiseが解決するまで待ちます。

```typescript
async function fetchDashboardData() {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),    // Promise<User[]>
    fetchPosts(),    // Promise<Post[]>
    fetchComments(), // Promise<Comment[]>
  ]);

  // users: User[], posts: Post[], comments: Comment[]
  return { users, posts, comments };
}
```

### Promise.allSettled

すべてのPromiseの結果（成功/失敗）を取得します。

```typescript
async function fetchMultipleResources() {
  const results = await Promise.allSettled([
    fetch("/api/users"),
    fetch("/api/posts"),
    fetch("/api/comments"),
  ]);

  // results: PromiseSettledResult<Response>[]
  results.forEach((result, index) => {
    if (result.status === "fulfilled") {
      console.log(`Request ${index}: Success`, result.value);
    } else {
      console.log(`Request ${index}: Failed`, result.reason);
    }
  });
}
```

### Promise.race

最初に完了したPromiseの結果を返します。

```typescript
async function fetchWithFallback<T>(
  primary: () => Promise<T>,
  fallback: () => Promise<T>,
  timeoutMs: number
): Promise<T> {
  try {
    return await Promise.race([
      primary(),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout")), timeoutMs)
      ),
    ]);
  } catch {
    return fallback();
  }
}
```

## ジェネリックな非同期パターン

### 型安全なAPIクライアント

```typescript
interface ApiEndpoints {
  "/users": User[];
  "/users/:id": User;
  "/posts": Post[];
  "/posts/:id": Post;
}

async function apiGet<T extends keyof ApiEndpoints>(
  endpoint: T
): Promise<ApiEndpoints[T]> {
  const response = await fetch(endpoint);
  return response.json();
}

// 使用例
const users = await apiGet("/users");     // User[]
const user = await apiGet("/users/:id");  // User
```

### リトライ機構

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries: number;
    delay: number;
    backoff?: number;
  }
): Promise<T> {
  const { maxRetries, delay, backoff = 2 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const waitTime = delay * Math.pow(backoff, attempt);
      console.log(`Attempt ${attempt + 1} failed, retrying in ${waitTime}ms...`);
      await new Promise((resolve) => setTimeout(resolve, waitTime));
    }
  }

  throw new Error("Unreachable");
}

// 使用例
const data = await withRetry(
  () => fetch("/api/data").then((r) => r.json()),
  { maxRetries: 3, delay: 1000 }
);
```

## AsyncIterable と for await...of

### 非同期イテレータ

```typescript
async function* generateNumbers(count: number): AsyncGenerator<number> {
  for (let i = 0; i < count; i++) {
    await new Promise((resolve) => setTimeout(resolve, 100));
    yield i;
  }
}

async function main() {
  for await (const num of generateNumbers(5)) {
    console.log(num); // 0, 1, 2, 3, 4（100msごと）
  }
}
```

### ページネーション

```typescript
async function* fetchAllPages<T>(
  fetchPage: (page: number) => Promise<{ data: T[]; hasNext: boolean }>
): AsyncGenerator<T[]> {
  let page = 1;
  let hasNext = true;

  while (hasNext) {
    const result = await fetchPage(page);
    yield result.data;
    hasNext = result.hasNext;
    page++;
  }
}

// 使用例
async function getAllUsers() {
  const allUsers: User[] = [];

  for await (const page of fetchAllPages<User>((p) =>
    fetch(`/api/users?page=${p}`).then((r) => r.json())
  )) {
    allUsers.push(...page);
  }

  return allUsers;
}
```

## Awaited 型

Promise をアンラップして中身の型を取得します。

```typescript
type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number
type C = Awaited<string | Promise<boolean>>; // string | boolean

// 関数の戻り値型と組み合わせ
async function fetchData() {
  return { users: [], count: 0 };
}

type Data = Awaited<ReturnType<typeof fetchData>>;
// { users: never[]; count: number }
```

## AbortController による型安全なキャンセル

```typescript
async function fetchWithCancel<T>(
  url: string,
  signal?: AbortSignal
): Promise<T> {
  const response = await fetch(url, { signal });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  return response.json() as Promise<T>;
}

// 使用例
const controller = new AbortController();

// 5秒後に自動キャンセル
setTimeout(() => controller.abort(), 5000);

try {
  const data = await fetchWithCancel<User[]>("/api/users", controller.signal);
  console.log(data);
} catch (error) {
  if (error instanceof DOMException && error.name === "AbortError") {
    console.log("リクエストがキャンセルされました");
  }
}
```

## まとめ

- `Promise<T>` でPromiseの解決値の型を指定する
- async関数は自動的に `Promise<T>` を返す
- `Promise.all` の結果はタプル型として推論される
- Result型パターンでエラーハンドリングを型安全にできる
- AsyncGenerator で非同期イテレーションが可能
- `Awaited<T>` でPromiseをアンラップした型を取得できる

次回は「React × TypeScript」について詳しく解説します。
