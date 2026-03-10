# TypeScript 5.x の最新機能まとめ

## はじめに

TypeScriptは定期的にアップデートされ、新しい機能や改善が追加されています。この記事では、TypeScript 5.0から5.x系で追加された注目の機能をまとめて解説します。

## TypeScript 5.0

### ECMAScript標準デコレータ

Stage 3に到達したデコレータ提案がサポートされました。`experimentalDecorators` フラグなしで使えます。

```typescript
function log(
  target: Function,
  context: ClassMethodDecoratorContext
) {
  return function (this: any, ...args: any[]) {
    console.log(`Calling ${String(context.name)} with`, args);
    return (target as Function).apply(this, args);
  };
}

class Calculator {
  @log
  add(a: number, b: number) {
    return a + b;
  }
}
```

### const 型パラメータ

ジェネリック関数の型パラメータに `const` 修飾子を付けることで、渡された値をリテラル型として推論させることができます。

```typescript
// const なし
function withoutConst<T extends readonly string[]>(args: T): T {
  return args;
}
const a = withoutConst(["hello", "world"]); // readonly string[]

// const あり
function withConst<const T extends readonly string[]>(args: T): T {
  return args;
}
const b = withConst(["hello", "world"]); // readonly ["hello", "world"]
```

これは `as const` を引数側で書く代わりに、関数定義側で指定できる便利な機能です。

```typescript
// ルーター定義の例
function defineRoutes<const T extends Record<string, string>>(routes: T): T {
  return routes;
}

const routes = defineRoutes({
  home: "/",
  about: "/about",
  users: "/users",
});

// routes の型: { readonly home: "/"; readonly about: "/about"; readonly users: "/users" }
type RouteName = keyof typeof routes; // "home" | "about" | "users"
```

### すべてのenumがユニオンenumに

TypeScript 5.0以降、すべてのenumはユニオン型として扱われます。

```typescript
enum Color {
  Red,
  Green,
  Blue,
}

// Color は Color.Red | Color.Green | Color.Blue として扱われる

function paint(color: Color) {
  switch (color) {
    case Color.Red:
      return "赤";
    case Color.Green:
      return "緑";
    case Color.Blue:
      return "青";
    // 網羅性チェックが自動的に働く
  }
}
```

### moduleResolution: "bundler"

バンドラー（Vite、webpack、esbuild等）を使うプロジェクト向けの新しいモジュール解決戦略です。

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"
  }
}
```

特徴：
- 拡張子なしのインポートを許可
- `package.json` の `exports` / `imports` フィールドをサポート
- 相対インポートに `.js` 拡張子が不要

## TypeScript 5.1

### undefined を返す関数の戻り値型を省略可能に

```typescript
// 5.1以前は void が必要だった
function logAndReturn(message: string): undefined {
  console.log(message);
  return undefined;
}

// 5.1以降は return を省略できる
function logAndReturn2(message: string): undefined {
  console.log(message);
  // return 文なしでOK
}
```

### JSXの型チェック改善

React Server Components（RSC）のために、async関数コンポーネントの型チェックが改善されました。

```typescript
// async コンポーネントが可能に（RSC向け）
async function UserProfile({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  return <div>{user.name}</div>;
}
```

## TypeScript 5.2

### using 宣言（Explicit Resource Management）

`using` キーワードによるリソースの自動解放がサポートされました。

```typescript
// Symbol.dispose を実装したオブジェクト
class FileHandle {
  constructor(private path: string) {
    console.log(`Opening ${path}`);
  }

  read(): string {
    return "file contents";
  }

  [Symbol.dispose]() {
    console.log(`Closing ${this.path}`);
  }
}

function processFile() {
  using file = new FileHandle("/path/to/file");
  const content = file.read();
  console.log(content);
  // スコープを抜けると自動的に [Symbol.dispose]() が呼ばれる
}
```

### 非同期版: await using

```typescript
class DatabaseConnection {
  static async connect(url: string) {
    console.log(`Connecting to ${url}`);
    return new DatabaseConnection(url);
  }

  constructor(private url: string) {}

  async query(sql: string): Promise<unknown[]> {
    return [];
  }

  async [Symbol.asyncDispose]() {
    console.log(`Disconnecting from ${this.url}`);
  }
}

async function fetchData() {
  await using db = await DatabaseConnection.connect("postgres://localhost");
  const users = await db.query("SELECT * FROM users");
  return users;
  // 自動的に接続が閉じられる
}
```

### Decorator Metadata

デコレータにメタデータ機能が追加されました。

```typescript
const METADATA_KEY = Symbol("metadata");

function tracked(
  target: any,
  context: ClassFieldDecoratorContext
) {
  context.metadata[METADATA_KEY] ??= [];
  (context.metadata[METADATA_KEY] as string[]).push(String(context.name));
}

class User {
  @tracked name: string = "";
  @tracked email: string = "";
  age: number = 0;
}

// メタデータからトラッキング対象のフィールドを取得
const trackedFields = User[Symbol.metadata]?.[METADATA_KEY];
// ["name", "email"]
```

## TypeScript 5.3

### Import Attributes

インポート時に属性を指定できます。

```typescript
import config from "./config.json" with { type: "json" };
```

### switch(true) のナローイング改善

```typescript
function processValue(value: string | number | boolean) {
  switch (true) {
    case typeof value === "string":
      console.log(value.toUpperCase()); // string として扱える
      break;
    case typeof value === "number":
      console.log(value.toFixed(2)); // number として扱える
      break;
    case typeof value === "boolean":
      console.log(value ? "true" : "false"); // boolean として扱える
      break;
  }
}
```

## TypeScript 5.4

### NoInfer ユーティリティ型

型推論の対象から特定の型パラメータを除外します。

```typescript
// NoInfer なし
function createFSM<S extends string>(config: {
  initial: S;
  states: Record<S, { on: Record<string, S> }>;
}) {}

// NoInfer あり - initial は states のキーから推論されない
function createFSM2<S extends string>(config: {
  initial: NoInfer<S>;
  states: Record<S, { on: Record<string, NoInfer<S>> }>;
}) {}

createFSM2({
  initial: "idle",
  states: {
    idle: { on: { start: "running" } },
    running: { on: { stop: "idle" } },
    // initial に "typo" と書いてもエラーになる
  },
});
```

### クロージャでのナローイング保持

```typescript
function processItems(items: string[] | undefined) {
  if (!items) return;

  // 5.4以降、クロージャ内でもナローイングが保持される
  const lengths = items.map((item) => {
    // items は string[] として扱える
    return item.length;
  });
}
```

## TypeScript 5.5

### 推論される型述語（Inferred Type Predicates）

配列のfilterメソッドなどで、自動的に型が絞り込まれるようになりました。

```typescript
const values = [1, null, 2, undefined, 3];

// 5.5以降、自動的に number[] と推論される
const numbers = values.filter((v) => v != null);
// 以前は (number | null | undefined)[] だった
```

### 正規表現の構文チェック

正規表現リテラルの構文がコンパイル時にチェックされるようになりました。

```typescript
const regex = /[a-z/; // コンパイルエラー！閉じ括弧がない
```

## 移行のヒント

### 段階的なアップグレード

```json
// まず strict を有効化
{
  "compilerOptions": {
    "strict": true
  }
}

// 次に追加の厳密チェックを有効化
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### 互換性の確認

```bash
# TypeScriptのバージョンを確認
npx tsc --version

# アップグレード
npm install typescript@latest --save-dev

# 型チェックのみ実行（コンパイルなし）
npx tsc --noEmit
```

## 今後の注目機能

TypeScriptチームが検討中・開発中の機能：

- **パターンマッチング** - `match` 式の提案
- **パイプライン演算子** - `|>` の型サポート
- **不変コレクション** - 組み込みのイミュータブル型
- **型レベルの算術** - 数値リテラル型の演算

## シリーズのまとめ

この20回のシリーズでは、TypeScriptの基礎から最新機能まで幅広くカバーしました。

1. **基本型** - プリミティブ型、配列、タプル
2. **型推論** - TypeScriptの賢い推論能力
3. **インターフェースと型エイリアス** - オブジェクト型の定義
4. **ジェネリクス** - 柔軟で型安全なコード
5. **ユニオンとインターセクション** - 型の合成
6. **型ガード** - ナローイングの仕組み
7. **keyof と typeof** - 型レベルの演算子
8. **Mapped Types** - 型の変換
9. **Conditional Types** - 型レベルの条件分岐
10. **Template Literal Types** - 文字列型の操作
11. **ユーティリティ型** - 組み込みの型ツール
12. **enum** - 列挙型と代替パターン
13. **モジュール** - コードの分割と管理
14. **デコレータ** - メタプログラミング
15. **非同期処理の型** - Promise と async/await
16. **React × TypeScript** - フロントエンド開発
17. **コンパイラオプション** - tsconfig.json の設定
18. **エラーハンドリング** - 型安全なエラー処理
19. **デザインパターン** - 設計パターンの実装
20. **TypeScript 5.x** - 最新機能のまとめ

TypeScriptは進化を続けています。公式ドキュメントやリリースノートをチェックしながら、最新の機能を活用していきましょう。
