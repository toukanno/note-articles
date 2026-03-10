# TypeScript - インターフェースと型エイリアスを使い分ける

## はじめに

TypeScriptでオブジェクトの型を定義するには、`interface` と `type`（型エイリアス）の2つの方法があります。どちらも似たことができますが、それぞれに特徴があります。

この記事では、両者の違いと使い分けについて解説します。

## interface の基本

`interface` はオブジェクトの形状（shape）を定義します。

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

const user: User = {
  name: "太郎",
  age: 25,
  email: "taro@example.com",
};
```

### メソッドの定義

```typescript
interface Calculator {
  // メソッドシグネチャ
  add(a: number, b: number): number;
  subtract(a: number, b: number): number;

  // プロパティとしての関数型
  multiply: (a: number, b: number) => number;
}
```

### インデックスシグネチャ

動的なキーを持つオブジェクトの型を定義できます。

```typescript
interface Dictionary {
  [key: string]: string;
}

const translations: Dictionary = {
  hello: "こんにちは",
  goodbye: "さようなら",
};
```

## type（型エイリアス）の基本

`type` は任意の型に名前をつけることができます。

```typescript
type User = {
  name: string;
  age: number;
  email: string;
};

// プリミティブ型にも名前をつけられる
type ID = string | number;

// タプルにも名前をつけられる
type Point = [number, number];

// 関数型にも名前をつけられる
type Formatter = (value: string) => string;
```

## 継承と拡張

### interface の extends

```typescript
interface Animal {
  name: string;
  age: number;
}

interface Dog extends Animal {
  breed: string;
  bark(): void;
}

// 複数のインターフェースを継承
interface GuideDog extends Dog {
  owner: string;
  guide(): void;
}
```

### type のインターセクション（&）

```typescript
type Animal = {
  name: string;
  age: number;
};

type Dog = Animal & {
  breed: string;
  bark(): void;
};
```

### 相互に拡張可能

```typescript
// interface が type を拡張
type BaseConfig = {
  debug: boolean;
};

interface AppConfig extends BaseConfig {
  apiUrl: string;
}

// type が interface をインターセクション
interface BaseUser {
  name: string;
}

type AdminUser = BaseUser & {
  role: "admin";
  permissions: string[];
};
```

## interface の宣言マージ

`interface` の独自機能として「宣言マージ」があります。同じ名前のインターフェースを複数回宣言すると、自動的にマージされます。

```typescript
interface Window {
  myCustomProperty: string;
}

// 既存の Window インターフェースに
// myCustomProperty が追加される
window.myCustomProperty = "hello";
```

これはライブラリの型定義を拡張するときに特に便利です。

```typescript
// express の Request を拡張
declare module "express" {
  interface Request {
    userId?: string;
  }
}
```

`type` では宣言マージはできません。同じ名前で再定義するとエラーになります。

```typescript
type User = { name: string };
// type User = { age: number }; // エラー！重複した識別子
```

## interface と type の違いまとめ

| 機能 | interface | type |
|---|---|---|
| オブジェクト型の定義 | OK | OK |
| 継承・拡張 | `extends` | `&`（インターセクション） |
| プリミティブ型の定義 | 不可 | OK |
| ユニオン型の定義 | 不可 | OK |
| タプル型の定義 | 不可 | OK |
| 宣言マージ | OK | 不可 |
| implements | OK | OK（一部制限あり） |

## 実践：使い分けガイド

### interface を使うべき場面

**1. クラスが実装する型を定義するとき**

```typescript
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

class UserModel implements Serializable {
  serialize(): string {
    return JSON.stringify(this);
  }
  deserialize(data: string): void {
    Object.assign(this, JSON.parse(data));
  }
}
```

**2. ライブラリの型定義を拡張する可能性があるとき**

```typescript
// サードパーティの型を拡張できるように interface で定義
interface PluginOptions {
  name: string;
  version: string;
}
```

**3. オブジェクトの形状を定義するとき（一般的なケース）**

```typescript
interface ApiResponse {
  status: number;
  data: unknown;
  message: string;
}
```

### type を使うべき場面

**1. ユニオン型を定義するとき**

```typescript
type Status = "pending" | "active" | "inactive";
type Result = Success | Failure;
```

**2. 関数型を定義するとき**

```typescript
type EventHandler = (event: Event) => void;
type AsyncAction<T> = () => Promise<T>;
```

**3. 複雑な型の組み合わせを定義するとき**

```typescript
type Nullable<T> = T | null;
type ReadonlyUser = Readonly<User>;
type UserKeys = keyof User;
```

**4. タプル型を定義するとき**

```typescript
type Coordinate = [number, number];
type NameAge = [string, number];
```

## 高度なパターン

### Discriminated Union（判別可能なユニオン）

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}

interface Rectangle {
  kind: "rectangle";
  width: number;
  height: number;
}

interface Triangle {
  kind: "triangle";
  base: number;
  height: number;
}

type Shape = Circle | Rectangle | Triangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return (shape.base * shape.height) / 2;
  }
}
```

### ジェネリックインターフェース

```typescript
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

interface User {
  id: string;
  name: string;
  email: string;
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> {
    // 実装
    return null;
  }
  async findAll(): Promise<User[]> {
    return [];
  }
  async create(data: Omit<User, "id">): Promise<User> {
    return { id: "1", ...data };
  }
  async update(id: string, data: Partial<User>): Promise<User> {
    return { id, name: "", email: "", ...data };
  }
  async delete(id: string): Promise<void> {
    // 実装
  }
}
```

## まとめ

- `interface` はオブジェクトの形状定義に特化しており、宣言マージが可能
- `type` はより柔軟で、ユニオン型やタプル型なども定義できる
- オブジェクト型の定義にはどちらを使ってもよいが、プロジェクト内で統一することが大切
- 迷ったら `interface` を使い、`interface` でできないことは `type` を使う

次回は「ジェネリクス入門」について詳しく解説します。
