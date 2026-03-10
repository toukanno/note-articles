# TypeScript - Conditional Typesで型レベルのプログラミング

## はじめに

Conditional Types（条件付き型）は、型レベルで条件分岐を行う仕組みです。「この型がAを満たすならB、そうでなければC」という条件式を型レベルで記述できます。

型システムの中でも上級者向けの機能ですが、使いこなせるとライブラリの型定義で強力な表現が可能になります。

## 基本構文

```typescript
T extends U ? X : Y
```

- `T` が `U` に代入可能（サブタイプ）なら `X`
- そうでなければ `Y`

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
type C = IsString<"hello">; // true（"hello" は string のサブタイプ）
```

## 実用的な例

### Null除外

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;

type A = NonNullable<string | null>;       // string
type B = NonNullable<number | undefined>;  // number
type C = NonNullable<null | undefined>;    // never
```

### 配列の要素型を取得

```typescript
type ElementOf<T> = T extends (infer E)[] ? E : never;

type A = ElementOf<string[]>;    // string
type B = ElementOf<number[]>;    // number
type C = ElementOf<string>;      // never（配列ではない）
```

### Promise の中身を取得

```typescript
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number（再帰的に解決）
type C = Awaited<string>;                    // string（Promiseでなければそのまま）
```

## infer キーワード

`infer` は条件付き型の中で型変数を導入するためのキーワードです。マッチした型の一部を「推論」して取り出せます。

### 関数の戻り値型を取得

```typescript
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => string>;           // string
type B = MyReturnType<(x: number) => boolean>; // boolean
```

### 関数の引数型を取得

```typescript
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;

type A = MyParameters<(name: string, age: number) => void>;
// [name: string, age: number]
```

### コンストラクタの型を取得

```typescript
type InstanceOf<T> = T extends new (...args: any[]) => infer I ? I : never;

class User {
  constructor(public name: string) {}
}

type A = InstanceOf<typeof User>; // User
```

### 文字列パターンから型を抽出

```typescript
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type A = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"
```

## 分配条件付き型（Distributive Conditional Types）

型パラメータにユニオン型が渡された場合、条件付き型は各メンバーに対して個別に適用されます。

```typescript
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// string[] | number[]
// （string | number)[] ではない！

// 分配を防ぐにはタプルで包む
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type B = ToArrayNonDist<string | number>;
// (string | number)[]
```

### Exclude と Extract

分配の性質を利用した組み込みユーティリティ型です。

```typescript
// ユニオンから特定の型を除外
type MyExclude<T, U> = T extends U ? never : T;

type A = MyExclude<"a" | "b" | "c", "a">;      // "b" | "c"
type B = MyExclude<string | number | boolean, string>; // number | boolean

// ユニオンから特定の型を抽出
type MyExtract<T, U> = T extends U ? T : never;

type C = MyExtract<"a" | "b" | "c", "a" | "b">; // "a" | "b"
```

## 実践的なパターン

### 型安全なイベントシステム

```typescript
type EventConfig = {
  click: { x: number; y: number };
  keydown: { key: string; code: string };
  scroll: { position: number };
  resize: { width: number; height: number };
};

type EventHandler<T extends keyof EventConfig> = (
  data: EventConfig[T]
) => void;

type EventHandlerMap = {
  [K in keyof EventConfig]?: EventHandler<K>;
};
```

### 関数のオーバーロード型

```typescript
type Overloads<T> = T extends {
  (...args: infer A1): infer R1;
  (...args: infer A2): infer R2;
}
  ? [(...args: A1) => R1, (...args: A2) => R2]
  : never;
```

### 深い部分型

```typescript
type DeepPartial<T> = T extends object
  ? {
      [K in keyof T]?: DeepPartial<T[K]>;
    }
  : T;

interface Config {
  database: {
    host: string;
    port: number;
    pool: {
      min: number;
      max: number;
    };
  };
  cache: {
    ttl: number;
    enabled: boolean;
  };
}

type PartialConfig = DeepPartial<Config>;
// すべてのプロパティが再帰的にオプショナルになる

const override: PartialConfig = {
  database: {
    pool: {
      max: 20, // min は省略可能
    },
  },
  // cache は省略可能
};
```

### 型レベルの条件分岐で API レスポンスを切り替え

```typescript
type ApiMethod = "list" | "get" | "create" | "update" | "delete";

type ApiResponse<M extends ApiMethod, T> = M extends "list"
  ? { data: T[]; total: number; page: number }
  : M extends "get"
    ? { data: T }
    : M extends "create"
      ? { data: T; created: true }
      : M extends "update"
        ? { data: T; updated: true }
        : M extends "delete"
          ? { success: true }
          : never;

interface User {
  id: string;
  name: string;
}

type ListResponse = ApiResponse<"list", User>;
// { data: User[]; total: number; page: number }

type GetResponse = ApiResponse<"get", User>;
// { data: User }

type DeleteResponse = ApiResponse<"delete", User>;
// { success: true }
```

## Template Literal Types との組み合わせ

```typescript
type Getter<T extends string> = `get${Capitalize<T>}`;
type Setter<T extends string> = `set${Capitalize<T>}`;

type Accessors<T> = {
  [K in keyof T as Getter<string & K>]: () => T[K];
} & {
  [K in keyof T as Setter<string & K>]: (value: T[K]) => void;
};

interface State {
  name: string;
  count: number;
}

type StateAccessors = Accessors<State>;
// {
//   getName: () => string;
//   getCount: () => number;
//   setName: (value: string) => void;
//   setCount: (value: number) => void;
// }
```

## まとめ

- Conditional Types は `T extends U ? X : Y` の構文で型レベルの条件分岐を行う
- `infer` で型のパターンマッチングと部分的な型推論ができる
- 分配条件付き型はユニオン型の各メンバーに個別に適用される
- `Exclude`、`Extract`、`ReturnType` などの組み込み型は Conditional Types で実装されている
- 再帰的な Conditional Types でネスト構造の変換が可能

次回は「Template Literal Types」について詳しく解説します。
