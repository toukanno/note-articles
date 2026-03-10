# TypeScript - デコレータ入門

## はじめに

デコレータ（Decorators）は、クラスやそのメンバーに対して宣言的にメタデータを付与したり、振る舞いを変更したりする仕組みです。

TypeScript 5.0でECMAScript標準準拠のデコレータがサポートされました（Stage 3）。この記事では新しいデコレータ仕様を中心に解説します。

## デコレータの有効化

```json
// tsconfig.json
{
  "compilerOptions": {
    // TC39 Stage 3 デコレータ（TypeScript 5.0+）
    // 特別なフラグ不要（デフォルトで使用可能）

    // レガシーデコレータ（Angular等で使用）
    // "experimentalDecorators": true
  }
}
```

## クラスデコレータ

クラス全体に適用するデコレータです。

```typescript
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class BankAccount {
  balance: number = 0;

  deposit(amount: number) {
    this.balance += amount;
  }
}
```

### 実践：ログ付きクラス

```typescript
function withLogging<T extends new (...args: any[]) => any>(
  target: T,
  context: ClassDecoratorContext
) {
  return class extends target {
    constructor(...args: any[]) {
      console.log(`Creating instance of ${context.name}`);
      super(...args);
      console.log(`Instance created with args:`, args);
    }
  };
}

@withLogging
class UserService {
  constructor(private apiUrl: string) {}
}

const service = new UserService("https://api.example.com");
// Creating instance of UserService
// Instance created with args: ["https://api.example.com"]
```

## メソッドデコレータ

メソッドに適用するデコレータです。

### 実行時間計測

```typescript
function measure<T extends (...args: any[]) => any>(
  target: T,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: Parameters<T>): ReturnType<T> {
    const start = performance.now();
    const result = target.call(this, ...args);
    const end = performance.now();
    console.log(`${String(context.name)} took ${(end - start).toFixed(2)}ms`);
    return result;
  } as T;
}

class DataProcessor {
  @measure
  processData(data: number[]): number {
    return data.reduce((sum, n) => sum + n, 0);
  }
}
```

### メソッドのバリデーション

```typescript
function validateArgs(
  target: Function,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    for (const arg of args) {
      if (arg === null || arg === undefined) {
        throw new Error(
          `${String(context.name)}: null/undefined argument detected`
        );
      }
    }
    return (target as Function).apply(this, args);
  };
}

class UserRepository {
  @validateArgs
  findById(id: string) {
    // id が null/undefined なら例外が投げられる
    return { id, name: "太郎" };
  }
}
```

### デバウンス

```typescript
function debounce(ms: number) {
  return function <T extends (...args: any[]) => any>(
    target: T,
    context: ClassMethodDecoratorContext
  ) {
    let timeoutId: ReturnType<typeof setTimeout>;

    return function (this: any, ...args: Parameters<T>) {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => {
        target.apply(this, args);
      }, ms);
    } as unknown as T;
  };
}

class SearchComponent {
  @debounce(300)
  onSearch(query: string) {
    console.log(`Searching for: ${query}`);
  }
}
```

## フィールドデコレータ

クラスフィールドに適用するデコレータです。

```typescript
function min(minValue: number) {
  return function (
    target: undefined,
    context: ClassFieldDecoratorContext
  ) {
    return function (initialValue: number) {
      if (initialValue < minValue) {
        throw new Error(
          `${String(context.name)} must be at least ${minValue}`
        );
      }
      return initialValue;
    };
  };
}

function max(maxValue: number) {
  return function (
    target: undefined,
    context: ClassFieldDecoratorContext
  ) {
    return function (initialValue: number) {
      if (initialValue > maxValue) {
        throw new Error(
          `${String(context.name)} must be at most ${maxValue}`
        );
      }
      return initialValue;
    };
  };
}

class Config {
  @min(1)
  @max(100)
  maxRetries = 3;
}
```

## アクセサデコレータ

getter/setter に適用するデコレータです。

```typescript
function logged<T>(
  target: ClassAccessorDecoratorTarget<any, T>,
  context: ClassAccessorDecoratorContext
): ClassAccessorDecoratorResult<any, T> {
  return {
    get(this: any) {
      const value = target.get.call(this);
      console.log(`Getting ${String(context.name)}: ${value}`);
      return value;
    },
    set(this: any, value: T) {
      console.log(`Setting ${String(context.name)} to: ${value}`);
      target.set.call(this, value);
    },
  };
}

class User {
  @logged
  accessor name: string = "太郎";
}

const user = new User();
user.name;          // Getting name: 太郎
user.name = "花子"; // Setting name to: 花子
```

## デコレータファクトリ

引数を受け取るデコレータは「デコレータファクトリ」パターンで実装します。

```typescript
function retry(maxAttempts: number, delayMs: number = 1000) {
  return function <T extends (...args: any[]) => Promise<any>>(
    target: T,
    context: ClassMethodDecoratorContext
  ) {
    return async function (this: any, ...args: Parameters<T>) {
      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          return await target.apply(this, args);
        } catch (error) {
          if (attempt === maxAttempts) throw error;
          console.log(
            `${String(context.name)} failed (attempt ${attempt}/${maxAttempts}), retrying...`
          );
          await new Promise((resolve) => setTimeout(resolve, delayMs));
        }
      }
    } as unknown as T;
  };
}

class ApiClient {
  @retry(3, 2000)
  async fetchData(url: string): Promise<unknown> {
    const response = await fetch(url);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  }
}
```

## 複数のデコレータの適用

デコレータは下から上に適用されます（内側から外側）。

```typescript
function first(
  target: Function,
  context: ClassMethodDecoratorContext
) {
  console.log("first decorator evaluated");
  return function (this: any, ...args: any[]) {
    console.log("first decorator executed");
    return (target as Function).apply(this, args);
  };
}

function second(
  target: Function,
  context: ClassMethodDecoratorContext
) {
  console.log("second decorator evaluated");
  return function (this: any, ...args: any[]) {
    console.log("second decorator executed");
    return (target as Function).apply(this, args);
  };
}

class Example {
  @first
  @second
  method() {
    console.log("method executed");
  }
}

// 評価順: second → first
// 実行順: first → second → method
```

## レガシーデコレータ vs 新デコレータ

| 特徴 | レガシー | 新（Stage 3） |
|---|---|---|
| tsconfig フラグ | `experimentalDecorators` | 不要 |
| パラメータデコレータ | あり | なし |
| メタデータ | `reflect-metadata` | なし（将来対応予定） |
| Angular 対応 | 必須 | 移行中 |
| NestJS 対応 | 必須 | 移行中 |

## まとめ

- デコレータはクラスやそのメンバーの振る舞いを宣言的に変更する
- TypeScript 5.0以降、Stage 3デコレータが利用可能
- クラス、メソッド、フィールド、アクセサに適用できる
- デコレータファクトリで引数を受け取れる
- 複数のデコレータは下から上に適用される
- Angular/NestJSではレガシーデコレータが依然として必要

次回は「非同期処理の型付け」について詳しく解説します。
