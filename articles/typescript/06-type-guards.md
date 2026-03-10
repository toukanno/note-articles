# TypeScript - 型ガードとナローイングを極める

## はじめに

TypeScriptでユニオン型を扱う場合、変数がどの型なのかを実行時に判定し、型を絞り込む必要があります。この仕組みを「ナローイング（Narrowing）」、そのための判定処理を「型ガード（Type Guard）」と呼びます。

## typeof 型ガード

最も基本的な型ガードです。プリミティブ型の判定に使います。

```typescript
function padLeft(value: string | number, padding: string | number): string {
  if (typeof padding === "number") {
    // padding は number に絞り込まれる
    return " ".repeat(padding) + value;
  }
  // padding は string に絞り込まれる
  return padding + value;
}
```

`typeof` で判定できる型：
- `"string"`
- `"number"`
- `"boolean"`
- `"bigint"`
- `"symbol"`
- `"undefined"`
- `"object"`（null も含む点に注意）
- `"function"`

## instanceof 型ガード

クラスのインスタンスかどうかを判定します。

```typescript
class HttpError {
  constructor(
    public status: number,
    public message: string
  ) {}
}

class ValidationError {
  constructor(
    public fields: string[],
    public message: string
  ) {}
}

function handleError(error: HttpError | ValidationError) {
  if (error instanceof HttpError) {
    // error は HttpError に絞り込まれる
    console.log(`HTTP ${error.status}: ${error.message}`);
  } else {
    // error は ValidationError に絞り込まれる
    console.log(`Validation failed: ${error.fields.join(", ")}`);
  }
}
```

## in 演算子による型ガード

オブジェクトが特定のプロパティを持つかどうかで判定します。

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    // animal は Fish に絞り込まれる
    animal.swim();
  } else {
    // animal は Bird に絞り込まれる
    animal.fly();
  }
}
```

## 等値チェックによるナローイング

`===`、`!==`、`==`、`!=` による比較でもナローイングが行われます。

```typescript
type Shape = "circle" | "square" | "triangle";

function getAngles(shape: Shape): number {
  if (shape === "circle") {
    return 0; // shape は "circle"
  }
  if (shape === "square") {
    return 4; // shape は "square"
  }
  return 3; // shape は "triangle"
}
```

### null / undefined チェック

```typescript
function processValue(value: string | null | undefined) {
  if (value == null) {
    // value は null | undefined（== null で両方チェック可能）
    return "値なし";
  }
  // value は string に絞り込まれる
  return value.toUpperCase();
}
```

## truthiness によるナローイング

```typescript
function printName(name: string | null | undefined) {
  if (name) {
    // name は string に絞り込まれる（ただし空文字は除外される点に注意）
    console.log(name.toUpperCase());
  }
}
```

**注意：** 空文字 `""` や `0` も falsy なので、意図しない絞り込みに注意が必要です。

## ユーザー定義型ガード

`is` キーワードを使って、カスタム型ガード関数を定義できます。

### 基本的な使い方

```typescript
interface Cat {
  type: "cat";
  meow(): void;
}

interface Dog {
  type: "dog";
  bark(): void;
}

// 型ガード関数
function isCat(animal: Cat | Dog): animal is Cat {
  return animal.type === "cat";
}

function handleAnimal(animal: Cat | Dog) {
  if (isCat(animal)) {
    animal.meow(); // Cat として使える
  } else {
    animal.bark(); // Dog として使える
  }
}
```

### 実践的な型ガード

```typescript
// null / undefined を除外する型ガード
function isDefined<T>(value: T | null | undefined): value is T {
  return value !== null && value !== undefined;
}

const values: (string | null | undefined)[] = ["hello", null, "world", undefined];
const definedValues: string[] = values.filter(isDefined);
// ["hello", "world"]
```

```typescript
// 特定の型を判定する汎用型ガード
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number";
}

function processInput(input: unknown) {
  if (isString(input)) {
    console.log(input.toUpperCase());
  } else if (isNumber(input)) {
    console.log(input.toFixed(2));
  }
}
```

### 配列要素の型ガード

```typescript
type Result = { status: "success"; data: string } | { status: "error"; error: string };

function isSuccess(result: Result): result is { status: "success"; data: string } {
  return result.status === "success";
}

const results: Result[] = [
  { status: "success", data: "データ1" },
  { status: "error", error: "エラー" },
  { status: "success", data: "データ2" },
];

const successResults = results.filter(isSuccess);
// { status: "success"; data: string }[]
successResults.forEach((r) => console.log(r.data)); // 安全にアクセス可能
```

## アサーション関数（asserts）

`asserts` キーワードを使うと、条件を満たさない場合に例外を投げる関数を型安全に定義できます。

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function processValue(value: unknown) {
  assertIsString(value);
  // この行以降、value は string 型として扱える
  console.log(value.toUpperCase());
}
```

```typescript
// null チェック用のアサーション関数
function assertDefined<T>(
  value: T | null | undefined,
  message?: string
): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(message ?? "Value is null or undefined");
  }
}

function getUser(id: string) {
  const user: User | null = findUser(id);
  assertDefined(user, `User ${id} not found`);
  // この行以降、user は User 型
  console.log(user.name);
}
```

## Discriminated Union のナローイング

```typescript
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" }
  | { type: "SET"; value: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT":
      return state + action.amount;
    case "DECREMENT":
      return state - action.amount;
    case "RESET":
      return 0;
    case "SET":
      return action.value;
  }
}
```

## 制御フロー解析

TypeScriptのコンパイラは、コードの制御フローを解析してナローイングを行います。

```typescript
function example(value: string | number | boolean) {
  if (typeof value === "string") {
    return value.toUpperCase(); // string
  }

  // ここでは string が除外される
  // value は number | boolean

  if (typeof value === "number") {
    return value.toFixed(2); // number
  }

  // ここでは number も除外される
  // value は boolean
  return value ? "true" : "false";
}
```

### 早期リターンパターン

```typescript
function processUser(user: User | null): string {
  if (!user) {
    return "ユーザーが見つかりません";
  }

  // この行以降、user は User 型（null が除外される）
  return `こんにちは、${user.name}さん`;
}
```

## まとめ

| 型ガード | 使用場面 |
|---|---|
| `typeof` | プリミティブ型の判定 |
| `instanceof` | クラスインスタンスの判定 |
| `in` | プロパティの存在チェック |
| `===` / `!==` | リテラル型・判別子の判定 |
| truthiness | null / undefined の簡易チェック |
| `is` (ユーザー定義) | カスタムロジックでの判定 |
| `asserts` | 条件を満たさない場合に例外 |

次回は「keyof と typeof 演算子」について詳しく解説します。
