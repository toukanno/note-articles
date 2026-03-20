# TypeScript - enumの使い方と注意点

## はじめに

`enum`（列挙型）は、関連する定数のセットに名前をつけて管理する機能です。しかし、TypeScriptの `enum` にはいくつかの落とし穴があり、代替手段が推奨される場面もあります。

この記事では、enumの使い方と注意点、そして代替パターンを解説します。

## 数値enum

最も基本的なenum。デフォルトで0から始まる数値が割り当てられます。

```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right, // 3
}

const dir: Direction = Direction.Up;
console.log(dir); // 0
```

### 値を明示的に指定

```typescript
enum HttpStatus {
  OK = 200,
  Created = 201,
  BadRequest = 400,
  Unauthorized = 401,
  NotFound = 404,
  InternalServerError = 500,
}

function handleResponse(status: HttpStatus) {
  if (status === HttpStatus.OK) {
    console.log("成功");
  }
}
```

## 文字列enum

各メンバーに文字列値を割り当てます。

```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

console.log(Color.Red); // "RED"
```

文字列enumの利点：
- デバッグ時に値が読みやすい
- 数値enumのような逆引きマッピングがない（余分なコードが生成されない）

## const enum

`const enum` はコンパイル時にインライン展開され、ランタイムにenum自体のオブジェクトが残りません。

```typescript
const enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

const dir = Direction.Up;
// コンパイル結果: const dir = "UP";
```

パフォーマンスは良いですが、いくつかの制約があります：
- `--isolatedModules` では使えない
- バンドラーによっては問題が起きることがある

## enumの注意点・落とし穴

### 1. 数値enumは型安全でない

```typescript
enum Status {
  Active = 0,
  Inactive = 1,
}

// 任意の number が代入できてしまう！
const status: Status = 999; // エラーにならない！
```

これは数値enumの大きな問題です。文字列enumではこの問題は発生しません。

### 2. 逆引きマッピングによるコード膨張

数値enumはコンパイル時に逆引きマッピングが生成されます。

```typescript
enum Direction {
  Up,
  Down,
}

// コンパイル結果：
// var Direction;
// (function (Direction) {
//   Direction[Direction["Up"] = 0] = "Up";
//   Direction[Direction["Down"] = 1] = "Down";
// })(Direction || (Direction = {}));
```

これにより、バンドルサイズが増加します。

### 3. Tree-shakingできない

enumはIIFE（即座に実行される関数式）としてコンパイルされるため、バンドラーがTree-shaking（未使用コード除去）できません。

### 4. ランタイムとコンパイル時の不一致

```typescript
enum Fruit {
  Apple = "APPLE",
  Banana = "BANANA",
}

// 文字列を直接渡すとエラー
// const fruit: Fruit = "APPLE"; // エラー！

// enum メンバー経由でのみ代入可能
const fruit: Fruit = Fruit.Apple;
```

## 代替パターン（推奨）

### パターン1: as const オブジェクト

最も推奨される代替手段です。

```typescript
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

type Direction = (typeof Direction)[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"

function move(direction: Direction) {
  console.log(`Moving ${direction}`);
}

move(Direction.Up);  // OK
move("UP");          // OK（文字列リテラルも受け入れる）
```

利点：
- Tree-shakingが効く
- 型安全
- バンドルサイズが小さい
- `--isolatedModules` でも問題ない
- JavaScriptとの相互運用性が高い

### パターン2: ユニオン型

単純なケースならユニオン型だけでも十分です。

```typescript
type Status = "active" | "inactive" | "pending";

function setStatus(status: Status) {
  // ...
}

setStatus("active"); // OK
// setStatus("unknown"); // エラー！
```

### パターン3: as const オブジェクト + ヘルパー関数

値の一覧や検証が必要な場合：

```typescript
const LogLevel = {
  Debug: 0,
  Info: 1,
  Warn: 2,
  Error: 3,
} as const;

type LogLevel = (typeof LogLevel)[keyof typeof LogLevel];

// 値の一覧を取得
const logLevels = Object.values(LogLevel);

// 値が有効かチェック
function isValidLogLevel(value: number): value is LogLevel {
  return logLevels.includes(value as LogLevel);
}
```

## enumを使ってもよい場面

以下の場面ではenumを使ってもよいでしょう：

1. **Angular** - AngularはenumをHTMLテンプレートで使えるようにサポートしている
2. **既存のコードベースで統一されている場合** - 一貫性のためにenumを継続使用
3. **数値のビットフラグ** - ビット演算を使う場面

```typescript
// ビットフラグの例
enum Permission {
  None = 0,
  Read = 1 << 0,    // 1
  Write = 1 << 1,   // 2
  Execute = 1 << 2, // 4
  All = Read | Write | Execute, // 7
}

const userPermission = Permission.Read | Permission.Write; // 3

if (userPermission & Permission.Read) {
  console.log("読み取り権限あり");
}
```

## 比較表

| 方法 | 型安全 | Tree-shaking | バンドルサイズ | ランタイム値 |
|---|---|---|---|---|
| 数値 enum | 低い | 不可 | 大 | あり |
| 文字列 enum | 高い | 不可 | 中 | あり |
| const enum | 高い | - (インライン) | 小 | なし |
| as const | 高い | 可能 | 小 | あり |
| ユニオン型 | 高い | - | なし | なし |

## まとめ

- 数値enumは型安全性に問題がある
- 文字列enumは比較的安全だが、Tree-shakingできない
- `as const` オブジェクトが最も推奨される代替手段
- シンプルなケースならユニオン型だけでも十分
- 既存プロジェクトの慣習に従うことも重要

次回は「TypeScriptのモジュールシステム」について詳しく解説します。
