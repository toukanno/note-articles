# TypeScriptの型推論を完全に理解する

## はじめに

TypeScriptの大きな特徴の一つが「型推論（Type Inference）」です。すべての変数に型を書かなくても、TypeScriptが自動的に型を推測してくれます。

型推論をうまく活用すれば、型の安全性を保ちながらも簡潔なコードが書けます。

## 基本的な型推論

### 変数の初期化

変数に値を代入すると、TypeScriptはその値から型を推論します。

```typescript
let message = "こんにちは"; // string と推論
let count = 42;             // number と推論
let isValid = true;         // boolean と推論

// 明示的に書くのと同じ効果
// let message: string = "こんにちは";
```

### const と let の推論の違い

`const` と `let` では推論される型が異なります。

```typescript
const status = "active";  // 型は "active"（リテラル型）
let status2 = "active";   // 型は string

const count = 42;          // 型は 42（リテラル型）
let count2 = 42;           // 型は number
```

`const` は再代入できないため、より具体的なリテラル型として推論されます。

## 関数の型推論

### 戻り値の推論

関数の戻り値の型は、return文から自動的に推論されます。

```typescript
// 戻り値は number と推論される
function add(a: number, b: number) {
  return a + b;
}

// 戻り値は string と推論される
function greet(name: string) {
  return `こんにちは、${name}さん`;
}

// 戻り値は boolean と推論される
function isAdult(age: number) {
  return age >= 18;
}
```

### コールバック関数の引数推論

配列メソッドなどのコールバックでは、引数の型も推論されます。

```typescript
const numbers = [1, 2, 3, 4, 5];

// num は number と推論される
const doubled = numbers.map((num) => num * 2);

// item は string と推論される
const fruits = ["りんご", "みかん", "バナナ"];
fruits.forEach((item) => {
  console.log(item.length); // string のメソッドが使える
});
```

## Best Common Type（最適共通型）

複数の値から型を推論する場合、TypeScriptは全ての値の型を包含する「最適共通型」を見つけます。

```typescript
// (number | string)[] と推論
const mixed = [1, "hello", 2, "world"];

// (number | null)[] と推論
const values = [1, 2, null, 4];
```

## Contextual Typing（文脈的型付け）

式が出現する「場所」に基づいて型が推論されることもあります。

```typescript
// イベントリスナーの例
document.addEventListener("click", (event) => {
  // event は MouseEvent と推論される
  console.log(event.clientX);
  console.log(event.clientY);
});

document.addEventListener("keydown", (event) => {
  // event は KeyboardEvent と推論される
  console.log(event.key);
});
```

### 型定義済みの変数への代入

```typescript
type User = {
  name: string;
  age: number;
};

// オブジェクトリテラルの各プロパティの型が推論される
const user: User = {
  name: "太郎",  // string であることが期待される
  age: 25,       // number であることが期待される
};
```

## 型推論の限界と明示的な型注釈

型推論が効かない場面では、明示的に型を書く必要があります。

### 遅延初期化

```typescript
// 初期値がないので型を推論できない
let result: string;

if (someCondition) {
  result = "成功";
} else {
  result = "失敗";
}
```

### 関数の引数

関数の引数は型推論されません（コールバック関数を除く）。

```typescript
// 引数には型注釈が必要
function calculateTax(price: number, taxRate: number): number {
  return price * taxRate;
}
```

### 空の配列

```typescript
// any[] と推論されるので、型を明示すべき
const items: string[] = [];
items.push("りんご");
```

## as const アサーション

`as const` を使うと、値全体をリテラル型として固定できます。

```typescript
// as const なし
const colors = ["red", "green", "blue"]; // string[]

// as const あり
const colors2 = ["red", "green", "blue"] as const;
// readonly ["red", "green", "blue"]

// オブジェクトにも使える
const config = {
  endpoint: "https://api.example.com",
  timeout: 3000,
} as const;
// { readonly endpoint: "https://api.example.com"; readonly timeout: 3000 }
```

## satisfies 演算子（TypeScript 4.9+）

`satisfies` を使うと、型チェックを行いながらも推論された型を保持できます。

```typescript
type Colors = Record<string, string | string[]>;

const palette = {
  red: "#ff0000",
  green: "#00ff00",
  blue: ["#0000ff", "#0000cc"],
} satisfies Colors;

// 推論された型が保持されるので、以下が可能
palette.red.toUpperCase();       // OK: string と推論
palette.blue.map((c) => c);     // OK: string[] と推論
```

`satisfies` がなかった場合、`palette.red` は `string | string[]` となり、直接 `toUpperCase()` を呼べません。

## 実践的なアドバイス

### 型注釈を書くべき場面

1. **関数の引数** - 常に書く
2. **公開APIの戻り値** - ライブラリや共有モジュールの関数
3. **複雑なオブジェクト** - 推論だけでは意図が伝わりにくい場合
4. **空の初期値** - `[]` や `{}` など

### 型注釈を省略してよい場面

1. **ローカル変数の初期化** - `const name = "太郎"` で十分
2. **関数の戻り値**（シンプルな場合）- 推論に任せる
3. **コールバック関数の引数** - 文脈から推論される

```typescript
// 良い例：必要な箇所だけ型を書く
function processUsers(users: User[]) {
  const names = users.map((u) => u.name);     // string[] と推論
  const adults = users.filter((u) => u.age >= 18); // User[] と推論
  return { names, adults }; // 戻り値も推論される
}
```

## まとめ

- TypeScriptは賢い型推論を持っており、多くの場面で型を自動推測する
- `const` はリテラル型、`let` はより広い型として推論される
- コールバック関数の引数は文脈から推論される
- `as const` で値をリテラル型として固定できる
- `satisfies` で型チェックと型推論を両立できる
- すべてに型を書く必要はないが、引数や公開APIには明示的に書こう

次回は「インターフェースと型エイリアス」について詳しく解説します。
