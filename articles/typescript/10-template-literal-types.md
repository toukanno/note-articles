# TypeScript - Template Literal Typesで文字列型を操る

## はじめに

Template Literal Types（テンプレートリテラル型）は、TypeScript 4.1で導入された機能です。JavaScriptのテンプレートリテラルと同じ構文を型レベルで使い、文字列型を合成・分解・変換できます。

## 基本構文

```typescript
type Greeting = `Hello, ${string}`;

const a: Greeting = "Hello, World";  // OK
const b: Greeting = "Hello, 太郎";   // OK
// const c: Greeting = "Hi, World";  // エラー！"Hello, "で始まらない
```

### リテラル型の組み合わせ

```typescript
type Color = "red" | "blue" | "green";
type Size = "small" | "medium" | "large";

type ColorSize = `${Color}-${Size}`;
// "red-small" | "red-medium" | "red-large"
// | "blue-small" | "blue-medium" | "blue-large"
// | "green-small" | "green-medium" | "green-large"
```

すべての組み合わせが自動的に生成されます。

## 組み込み文字列操作型

TypeScriptには4つの組み込み文字列操作型があります。

```typescript
type A = Uppercase<"hello">;     // "HELLO"
type B = Lowercase<"HELLO">;     // "hello"
type C = Capitalize<"hello">;    // "Hello"
type D = Uncapitalize<"Hello">;  // "hello"
```

### ユニオン型にも適用可能

```typescript
type Events = "click" | "scroll" | "keydown";
type HandlerNames = `on${Capitalize<Events>}`;
// "onClick" | "onScroll" | "onKeydown"
```

## 実践的なパターン

### CSSプロパティ名の型

```typescript
type CSSProperty = "margin" | "padding" | "border";
type Direction = "top" | "right" | "bottom" | "left";

type DirectionalCSS = `${CSSProperty}-${Direction}`;
// "margin-top" | "margin-right" | "margin-bottom" | "margin-left"
// | "padding-top" | ...
// | "border-top" | ...

type CSSValue = string;
type DirectionalStyles = Partial<Record<DirectionalCSS, CSSValue>>;

const styles: DirectionalStyles = {
  "margin-top": "10px",
  "padding-left": "20px",
};
```

### BEM記法の型

```typescript
type Block = "button" | "card" | "modal";
type Element = "title" | "body" | "footer";
type Modifier = "active" | "disabled" | "hidden";

type BEMBlock = Block;
type BEMElement = `${Block}__${Element}`;
type BEMModifier = `${Block}--${Modifier}` | `${BEMElement}--${Modifier}`;

type BEMClass = BEMBlock | BEMElement | BEMModifier;

const className: BEMClass = "button__title--active"; // OK
// const bad: BEMClass = "button_title";            // エラー！
```

### イベントハンドラーの型

```typescript
type DOMEvents = "click" | "focus" | "blur" | "change" | "submit";

type EventHandlerProps = {
  [K in DOMEvents as `on${Capitalize<K>}`]?: (event: Event) => void;
};

// 結果：
// {
//   onClick?: (event: Event) => void;
//   onFocus?: (event: Event) => void;
//   onBlur?: (event: Event) => void;
//   onChange?: (event: Event) => void;
//   onSubmit?: (event: Event) => void;
// }
```

### URLパスパラメータの抽出

```typescript
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
      ? Param
      : never;

type Params1 = ExtractParams<"/users/:userId">;
// "userId"

type Params2 = ExtractParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"

type Params3 = ExtractParams<"/about">;
// never

// パラメータオブジェクトの型を生成
type RouteParams<T extends string> = {
  [K in ExtractParams<T>]: string;
};

type UserPostParams = RouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }
```

### 型安全なパス文字列

```typescript
type PathImpl<T, K extends keyof T> = K extends string
  ? T[K] extends Record<string, unknown>
    ? K | `${K}.${PathImpl<T[K], keyof T[K]>}`
    : K
  : never;

type Path<T> = PathImpl<T, keyof T>;

interface FormData {
  user: {
    name: string;
    address: {
      city: string;
      zip: string;
    };
  };
  settings: {
    theme: string;
  };
}

type FormPaths = Path<FormData>;
// "user" | "user.name" | "user.address" | "user.address.city"
// | "user.address.zip" | "settings" | "settings.theme"
```

### SQLクエリビルダー風の型

```typescript
type Table = "users" | "posts" | "comments";
type OrderDirection = "ASC" | "DESC";

type SelectQuery = `SELECT * FROM ${Table}`;
type WhereClause = `WHERE ${string}`;
type OrderByClause = `ORDER BY ${string} ${OrderDirection}`;
type LimitClause = `LIMIT ${number}`;

type SimpleQuery =
  | SelectQuery
  | `${SelectQuery} ${WhereClause}`
  | `${SelectQuery} ${OrderByClause}`
  | `${SelectQuery} ${WhereClause} ${OrderByClause}`
  | `${SelectQuery} ${LimitClause}`;

const query1: SelectQuery = "SELECT * FROM users"; // OK
```

### 文字列の分割型

```typescript
type Split<
  S extends string,
  D extends string
> = S extends `${infer Head}${D}${infer Tail}`
  ? [Head, ...Split<Tail, D>]
  : [S];

type A = Split<"a.b.c", ".">;   // ["a", "b", "c"]
type B = Split<"hello", "">;     // ["h", "e", "l", "l", "o"]
type C = Split<"a-b-c", "-">;   // ["a", "b", "c"]
```

### ケバブケースからキャメルケースへの変換

```typescript
type KebabToCamel<S extends string> =
  S extends `${infer Head}-${infer Tail}`
    ? `${Head}${KebabToCamel<Capitalize<Tail>>}`
    : S;

type A = KebabToCamel<"background-color">;    // "backgroundColor"
type B = KebabToCamel<"border-top-width">;     // "borderTopWidth"
type C = KebabToCamel<"margin">;               // "margin"
```

### キャメルケースからケバブケースへの変換

```typescript
type CamelToKebab<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Head extends Uppercase<Head>
      ? `-${Lowercase<Head>}${CamelToKebab<Tail>}`
      : `${Head}${CamelToKebab<Tail>}`
    : S;

type A = CamelToKebab<"backgroundColor">;  // "background-color"
type B = CamelToKebab<"borderTopWidth">;   // "border-top-width"
```

## パフォーマンスの注意点

Template Literal Typesで大量の組み合わせを生成すると、コンパイル時間が大幅に増加する可能性があります。

```typescript
// 注意：組み合わせが爆発的に増える
type Huge = `${string}-${string}-${string}-${string}`;
// → コンパイラに大きな負荷がかかる可能性
```

必要最小限の組み合わせに留めることを推奨します。

## まとめ

- Template Literal Typesは文字列型をテンプレートリテラル構文で合成できる
- ユニオン型と組み合わせると、すべての組み合わせが自動生成される
- `Uppercase`、`Lowercase`、`Capitalize`、`Uncapitalize` で文字列型を変換できる
- `infer` と組み合わせてパターンマッチング・文字列分解ができる
- URLパラメータ抽出やケース変換など、実用的なパターンが多い

次回は「ユーティリティ型完全ガイド」について詳しく解説します。
