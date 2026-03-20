# TypeScript - Mapped Typesで型を変換する

## はじめに

Mapped Types（マップ型）は、既存の型を変換して新しい型を生成する仕組みです。型のプロパティを一括で変換できるため、重複のない型定義が可能になります。

TypeScriptの組み込みユーティリティ型（`Partial`、`Required`、`Readonly` など）も、内部的にはMapped Typesで実装されています。

## 基本構文

```typescript
type MappedType<T> = {
  [K in keyof T]: T[K];
};
```

- `K in keyof T` — T のすべてのキーに対して繰り返す
- `T[K]` — 各キーに対応する型

## 基本的なMapped Types

### すべてのプロパティをオプショナルにする

```typescript
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = MyPartial<User>;
// {
//   name?: string;
//   age?: number;
//   email?: string;
// }
```

### すべてのプロパティを必須にする

```typescript
type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};
```

`-?` はオプショナル修飾子を除去します。

### すべてのプロパティを読み取り専用にする

```typescript
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

type ReadonlyUser = MyReadonly<User>;
// {
//   readonly name: string;
//   readonly age: number;
//   readonly email: string;
// }
```

### 読み取り専用を解除する

```typescript
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};
```

`-readonly` で readonly 修飾子を除去します。

## Record 型

`Record` は指定したキーと値の型からオブジェクト型を生成します。

```typescript
type MyRecord<K extends string | number | symbol, V> = {
  [P in K]: V;
};

// 使用例
type PageInfo = {
  title: string;
  url: string;
};

type Pages = Record<"home" | "about" | "contact", PageInfo>;
// {
//   home: PageInfo;
//   about: PageInfo;
//   contact: PageInfo;
// }
```

## キーの変換（Key Remapping）

TypeScript 4.1以降、`as` を使ってキーを変換できます。

### キーにプレフィックスを追加

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// {
//   getName: () => string;
//   getAge: () => number;
// }
```

### 特定のキーを除外

```typescript
type RemoveKind<T> = {
  [K in keyof T as Exclude<K, "kind">]: T[K];
};

interface Shape {
  kind: "circle" | "square";
  radius?: number;
  width?: number;
}

type ShapeWithoutKind = RemoveKind<Shape>;
// {
//   radius?: number;
//   width?: number;
// }
```

### 値の型でフィルタリング

```typescript
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Mixed {
  name: string;
  age: number;
  email: string;
  isActive: boolean;
}

type StringProps = OnlyStrings<Mixed>;
// {
//   name: string;
//   email: string;
// }
```

## 実践的なパターン

### フォームのバリデーション型

```typescript
type ValidationErrors<T> = {
  [K in keyof T]?: string[];
};

interface SignUpForm {
  username: string;
  email: string;
  password: string;
  confirmPassword: string;
}

type SignUpErrors = ValidationErrors<SignUpForm>;
// {
//   username?: string[];
//   email?: string[];
//   password?: string[];
//   confirmPassword?: string[];
// }

const errors: SignUpErrors = {
  email: ["メールアドレスの形式が正しくありません"],
  password: ["8文字以上で入力してください", "数字を含めてください"],
};
```

### APIレスポンスのラッパー

```typescript
type ApiFields<T> = {
  [K in keyof T]: {
    value: T[K];
    isModified: boolean;
    error?: string;
  };
};

interface Product {
  name: string;
  price: number;
  description: string;
}

type ProductForm = ApiFields<Product>;
// {
//   name: { value: string; isModified: boolean; error?: string };
//   price: { value: number; isModified: boolean; error?: string };
//   description: { value: string; isModified: boolean; error?: string };
// }
```

### イベントリスナー型の生成

```typescript
type EventListeners<T> = {
  [K in keyof T as `on${Capitalize<string & K>}`]: (value: T[K]) => void;
};

interface StateChanges {
  name: string;
  age: number;
  active: boolean;
}

type StateListeners = EventListeners<StateChanges>;
// {
//   onName: (value: string) => void;
//   onAge: (value: number) => void;
//   onActive: (value: boolean) => void;
// }
```

### 深いReadonly

ネストされたオブジェクトも再帰的に読み取り専用にします。

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : DeepReadonly<T[K]>
    : T[K];
};

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
  server: {
    port: number;
  };
}

type ReadonlyConfig = DeepReadonly<Config>;
// すべてのプロパティが再帰的に readonly になる
```

### Pick と Omit の自作

```typescript
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type MyOmit<T, K extends keyof T> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type PublicUser = MyOmit<User, "password">;
// { id: number; name: string; email: string }

type Credentials = MyPick<User, "email" | "password">;
// { email: string; password: string }
```

## まとめ

- Mapped Typesは `[K in keyof T]` の構文で既存の型を変換する
- `?` / `-?` でオプショナルを追加・除去
- `readonly` / `-readonly` で読み取り専用を追加・除去
- `as` でキーのリマッピング（変換・フィルタリング）が可能
- 再帰的なMapped Typesで深いネスト構造にも対応できる
- 組み込みユーティリティ型の多くはMapped Typesで実装されている

次回は「Conditional Types」について詳しく解説します。
