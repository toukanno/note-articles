# TypeScript入門 - 基本的な型システムを理解しよう

## はじめに

TypeScriptはJavaScriptに「型」という概念を追加した言語です。型があることで、コードを実行する前にバグを発見でき、エディタの補完機能も格段に向上します。

この記事では、TypeScriptの基本的な型システムについて、実際のコード例とともに解説します。

## プリミティブ型

TypeScriptで最もよく使う基本的な型は以下の3つです。

### string（文字列）

```typescript
const name: string = "太郎";
const greeting: string = `こんにちは、${name}さん`;
```

### number（数値）

整数も浮動小数点数も `number` 型です。

```typescript
const age: number = 25;
const price: number = 1980.5;
const hex: number = 0xff;
```

### boolean（真偽値）

```typescript
const isActive: boolean = true;
const hasPermission: boolean = false;
```

## 配列の型

配列には2つの書き方があります。

```typescript
// 書き方1: 型名[]
const numbers: number[] = [1, 2, 3];
const names: string[] = ["太郎", "花子"];

// 書き方2: Array<型名>（ジェネリクス記法）
const scores: Array<number> = [85, 92, 78];
```

実務では `型名[]` の書き方が主流です。

## タプル型

要素の数と各要素の型が決まった配列を「タプル」として表現できます。

```typescript
// [名前, 年齢] のペア
const person: [string, number] = ["太郎", 25];

// 分割代入と組み合わせると便利
const [personName, personAge] = person;
console.log(personName); // "太郎"
console.log(personAge);  // 25
```

## オブジェクトの型

オブジェクトの型はプロパティごとに定義します。

```typescript
const user: { name: string; age: number; email: string } = {
  name: "太郎",
  age: 25,
  email: "taro@example.com",
};
```

### オプショナルプロパティ

`?` をつけると、そのプロパティは省略可能になります。

```typescript
const user: { name: string; age?: number } = {
  name: "太郎",
  // age は省略可能
};
```

### 読み取り専用プロパティ

`readonly` をつけると、再代入を禁止できます。

```typescript
const config: { readonly apiUrl: string; readonly timeout: number } = {
  apiUrl: "https://api.example.com",
  timeout: 3000,
};

// config.apiUrl = "other"; // エラー！readonlyなので変更不可
```

## 特殊な型

### any

どんな型の値でも受け入れます。型チェックが無効になるため、使用は極力避けましょう。

```typescript
let value: any = "hello";
value = 42;      // OK
value = true;    // OK
value.foo.bar;   // コンパイルエラーにならない（危険！）
```

### unknown

`any` と同様にどんな値も受け入れますが、使用する際に型チェックが必要です。外部からの入力を受け取るときに適しています。

```typescript
let value: unknown = "hello";

// そのままでは使えない
// console.log(value.length); // エラー！

// 型チェックしてから使う
if (typeof value === "string") {
  console.log(value.length); // OK
}
```

### void

関数が何も返さないことを表します。

```typescript
function logMessage(message: string): void {
  console.log(message);
}
```

### null と undefined

```typescript
const n: null = null;
const u: undefined = undefined;
```

`strictNullChecks` を有効にすると（推奨）、`null` や `undefined` を他の型に代入できなくなります。

### never

決して発生しない値を表します。例外を必ず投げる関数や、無限ループの関数の戻り値に使います。

```typescript
function throwError(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {
    // 永遠に終わらない
  }
}
```

## 型アサーション

コンパイラに「この値はこの型だ」と伝える方法です。

```typescript
const input = document.getElementById("name") as HTMLInputElement;
console.log(input.value);
```

ただし、型アサーションは型チェックを回避するため、誤用するとランタイムエラーの原因になります。本当に必要な場面でのみ使いましょう。

## リテラル型

特定の値だけを受け入れる型を定義できます。

```typescript
let direction: "north" | "south" | "east" | "west";
direction = "north"; // OK
// direction = "up"; // エラー！

let diceRoll: 1 | 2 | 3 | 4 | 5 | 6;
diceRoll = 3; // OK
// diceRoll = 7; // エラー！
```

## まとめ

TypeScriptの基本的な型をまとめると：

| 型 | 説明 | 例 |
|---|---|---|
| `string` | 文字列 | `"hello"` |
| `number` | 数値 | `42` |
| `boolean` | 真偽値 | `true` |
| `型名[]` | 配列 | `[1, 2, 3]` |
| `[型1, 型2]` | タプル | `["太郎", 25]` |
| `any` | 何でもOK（非推奨） | - |
| `unknown` | 安全なany | - |
| `void` | 戻り値なし | - |
| `never` | 発生しない値 | - |

次回は「型推論」について詳しく解説します。TypeScriptがいかに賢く型を推測してくれるかを見ていきましょう。
