# TypeScript - keyof と typeof 演算子で型を自在に操る

## はじめに

TypeScriptには、既存の型やオブジェクトから新しい型を導出するための強力な演算子があります。`keyof` と `typeof` はその中でも最もよく使うものです。

この記事では、これらの演算子の使い方と実践的なパターンを解説します。

## keyof 演算子

`keyof` はオブジェクト型のすべてのキーをユニオン型として取得します。

### 基本的な使い方

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type UserKeys = keyof User;
// "id" | "name" | "email" | "age"

const key: UserKeys = "name"; // OK
// const key2: UserKeys = "phone"; // エラー！
```

### 型安全なプロパティアクセス

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: 1, name: "太郎", email: "taro@example.com", age: 25 };

const name = getProperty(user, "name");   // string 型
const age = getProperty(user, "age");     // number 型
// getProperty(user, "phone");            // エラー！
```

### インデックスアクセス型（Indexed Access Types）

`T[K]` でオブジェクト型の特定のプロパティの型を取得できます。

```typescript
type UserName = User["name"]; // string
type UserId = User["id"];     // number

// ユニオンで複数指定も可能
type UserNameOrEmail = User["name" | "email"]; // string

// keyof と組み合わせ
type UserValues = User[keyof User]; // number | string
```

### 配列要素の型を取得

```typescript
const fruits = ["りんご", "みかん", "バナナ"] as const;
type Fruit = (typeof fruits)[number]; // "りんご" | "みかん" | "バナナ"
```

## typeof 演算子（型レベル）

JavaScriptの `typeof` とは異なり、TypeScriptの型レベルの `typeof` は変数や値から型を抽出します。

### 基本的な使い方

```typescript
const config = {
  apiUrl: "https://api.example.com",
  timeout: 3000,
  retries: 3,
  debug: false,
};

type Config = typeof config;
// {
//   apiUrl: string;
//   timeout: number;
//   retries: number;
//   debug: boolean;
// }
```

### 関数の型を取得

```typescript
function createUser(name: string, age: number) {
  return { id: Math.random().toString(), name, age };
}

type CreateUserFn = typeof createUser;
// (name: string, age: number) => { id: string; name: string; age: number }
```

### ReturnType と組み合わせ

```typescript
type UserResult = ReturnType<typeof createUser>;
// { id: string; name: string; age: number }

type CreateUserParams = Parameters<typeof createUser>;
// [name: string, age: number]
```

## keyof と typeof の組み合わせ

実務で非常によく使うパターンです。

### オブジェクトのキーからユニオン型を生成

```typescript
const STATUS = {
  PENDING: "pending",
  ACTIVE: "active",
  INACTIVE: "inactive",
} as const;

type StatusKey = keyof typeof STATUS;
// "PENDING" | "ACTIVE" | "INACTIVE"

type StatusValue = (typeof STATUS)[keyof typeof STATUS];
// "pending" | "active" | "inactive"
```

### 設定オブジェクトから型を導出

```typescript
const routes = {
  home: "/",
  about: "/about",
  users: "/users",
  userDetail: "/users/:id",
} as const;

type RouteName = keyof typeof routes;
// "home" | "about" | "users" | "userDetail"

type RoutePath = (typeof routes)[RouteName];
// "/" | "/about" | "/users" | "/users/:id"

function navigate(route: RouteName) {
  const path = routes[route];
  console.log(`Navigating to ${path}`);
}

navigate("home");    // OK
// navigate("login"); // エラー！
```

### enum の代替パターン

```typescript
const Color = {
  Red: "#ff0000",
  Green: "#00ff00",
  Blue: "#0000ff",
} as const;

type ColorName = keyof typeof Color;    // "Red" | "Green" | "Blue"
type ColorCode = (typeof Color)[ColorName]; // "#ff0000" | "#00ff00" | "#0000ff"

function setColor(name: ColorName) {
  const code = Color[name];
  document.body.style.backgroundColor = code;
}
```

## 実践的なパターン

### 型安全な設定マネージャー

```typescript
const defaultConfig = {
  theme: "light" as "light" | "dark",
  fontSize: 14,
  language: "ja" as "ja" | "en" | "zh",
  notifications: true,
};

type AppConfig = typeof defaultConfig;
type ConfigKey = keyof AppConfig;

class ConfigManager {
  private config: AppConfig;

  constructor() {
    this.config = { ...defaultConfig };
  }

  get<K extends ConfigKey>(key: K): AppConfig[K] {
    return this.config[key];
  }

  set<K extends ConfigKey>(key: K, value: AppConfig[K]): void {
    this.config[key] = value;
  }
}

const manager = new ConfigManager();
const theme = manager.get("theme");       // "light" | "dark"
manager.set("fontSize", 16);              // OK
// manager.set("fontSize", "large");      // エラー！number が必要
```

### 型安全なi18n

```typescript
const translations = {
  ja: {
    greeting: "こんにちは",
    farewell: "さようなら",
    thanks: "ありがとう",
  },
  en: {
    greeting: "Hello",
    farewell: "Goodbye",
    thanks: "Thank you",
  },
} as const;

type Locale = keyof typeof translations; // "ja" | "en"
type TranslationKey = keyof (typeof translations)["ja"];
// "greeting" | "farewell" | "thanks"

function t(locale: Locale, key: TranslationKey): string {
  return translations[locale][key];
}

t("ja", "greeting"); // "こんにちは"
// t("ja", "hello"); // エラー！"hello" は存在しないキー
```

### イベントハンドラーマップ

```typescript
const eventHandlers = {
  click: (x: number, y: number) => console.log(`Clicked at ${x}, ${y}`),
  keypress: (key: string) => console.log(`Pressed ${key}`),
  scroll: (position: number) => console.log(`Scrolled to ${position}`),
};

type EventName = keyof typeof eventHandlers;
type EventHandler<E extends EventName> = (typeof eventHandlers)[E];

function triggerEvent<E extends EventName>(
  event: E,
  ...args: Parameters<EventHandler<E>>
) {
  const handler = eventHandlers[event] as (...args: unknown[]) => void;
  handler(...args);
}

triggerEvent("click", 100, 200);    // OK
triggerEvent("keypress", "Enter");  // OK
// triggerEvent("click", "Enter");  // エラー！引数の型が違う
```

## まとめ

- `keyof` はオブジェクト型のキーをユニオン型として取得する
- `typeof`（型レベル）は値から型を抽出する
- `T[K]` でインデックスアクセス型を使い、プロパティの型を取得できる
- `keyof typeof` の組み合わせはオブジェクト定数から型を導出する定番パターン
- `as const` と組み合わせることで、リテラル型を保持した定数定義ができる

次回は「Mapped Types」について詳しく解説します。
