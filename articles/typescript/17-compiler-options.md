# TypeScriptのコンパイラオプション完全ガイド

## はじめに

`tsconfig.json` はTypeScriptプロジェクトの設定ファイルです。コンパイラオプションを適切に設定することで、型チェックの厳密さやコード生成の挙動を制御できます。

この記事では、重要なコンパイラオプションをカテゴリ別に解説します。

## 推奨設定（2024年版）

まず、一般的なプロジェクトでの推奨設定を示します。

```json
{
  "compilerOptions": {
    // 型チェック
    "strict": true,
    "noUncheckedIndexedAccess": true,

    // モジュール
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "isolatedModules": true,

    // 出力
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "outDir": "./dist",

    // パス
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },

    // その他
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## strict モード（型チェック系）

`"strict": true` は以下のオプションをまとめて有効にします。

### strictNullChecks

`null` と `undefined` を別の型として扱います。

```typescript
// strictNullChecks: true の場合
let name: string = "太郎";
// name = null;    // エラー！
// name = undefined; // エラー！

let maybeName: string | null = null; // OK
```

### strictFunctionTypes

関数の引数の型チェックを厳密にします。

```typescript
type Handler = (event: MouseEvent) => void;

// strictFunctionTypes: true の場合
const handler: Handler = (event: Event) => {}; // エラー！
// Event は MouseEvent のスーパータイプなので不可
```

### strictBindCallApply

`bind`、`call`、`apply` の引数の型チェックを行います。

```typescript
function add(a: number, b: number) {
  return a + b;
}

add.call(null, 1, 2);       // OK
// add.call(null, 1, "2");  // エラー！
```

### strictPropertyInitialization

クラスのプロパティがコンストラクタで初期化されていることを強制します。

```typescript
class User {
  name: string;
  // email: string; // エラー！初期化されていない

  constructor(name: string) {
    this.name = name;
  }
}
```

### noImplicitAny

暗黙の `any` を禁止します。

```typescript
// noImplicitAny: true の場合
// function greet(name) {}  // エラー！name の型が不明
function greet(name: string) {} // OK
```

### noImplicitThis

`this` の型が `any` になることを禁止します。

### useUnknownInCatchVariables

catch文の変数を `unknown` 型にします。

```typescript
try {
  // ...
} catch (error) {
  // useUnknownInCatchVariables: true の場合
  // error は unknown 型
  if (error instanceof Error) {
    console.log(error.message);
  }
}
```

## 追加の型チェックオプション

### noUncheckedIndexedAccess

インデックスアクセスの結果に `undefined` を含めます。

```typescript
const arr = [1, 2, 3];

// noUncheckedIndexedAccess: true の場合
const value = arr[0]; // number | undefined
if (value !== undefined) {
  console.log(value.toFixed(2)); // OK
}

const obj: Record<string, string> = {};
const name = obj["key"]; // string | undefined
```

### noUnusedLocals / noUnusedParameters

未使用のローカル変数・引数を警告します。

```typescript
// noUnusedLocals: true
function example() {
  const unused = "hello"; // エラー！未使用
  return 42;
}

// noUnusedParameters: true
function greet(name: string, unused: string) { // unused がエラー
  return `Hello, ${name}`;
}

// _ プレフィックスで回避
function greet(name: string, _unused: string) { // OK
  return `Hello, ${name}`;
}
```

### exactOptionalPropertyTypes

オプショナルプロパティに `undefined` を明示的に代入することを禁止します。

```typescript
interface User {
  name: string;
  age?: number;
}

// exactOptionalPropertyTypes: true の場合
const user1: User = { name: "太郎" };           // OK（age を省略）
// const user2: User = { name: "太郎", age: undefined }; // エラー！
```

## モジュール関連

### module

出力するモジュール形式を指定します。

```json
{
  "module": "ESNext"      // ESモジュール（推奨）
  // "module": "CommonJS"  // Node.js向け
  // "module": "NodeNext"  // Node.js ESM対応
}
```

### moduleResolution

モジュールの解決方法を指定します。

```json
{
  "moduleResolution": "bundler"  // Vite/webpack等のバンドラー使用時（推奨）
  // "moduleResolution": "node"    // 従来のNode.js方式
  // "moduleResolution": "nodenext" // Node.js ESM対応
}
```

### esModuleInterop

CommonJSモジュールのデフォルトインポートを許可します。

```typescript
// esModuleInterop: true の場合
import express from "express"; // OK

// esModuleInterop: false の場合
// import * as express from "express"; // この書き方が必要
```

### isolatedModules

各ファイルが独立してトランスパイルできることを保証します。Babelやesbuildなどのツールと併用する場合に重要です。

### resolveJsonModule

JSONファイルのインポートを許可します。

```typescript
import config from "./config.json";
// config の型が自動的に推論される
```

## 出力関連

### target

出力するJavaScriptのバージョンを指定します。

```json
{
  "target": "ES2022"  // 推奨：モダンブラウザ/Node.js向け
  // "target": "ES5"    // IE対応が必要な場合（レガシー）
  // "target": "ESNext" // 最新の機能をすべて使用
}
```

### lib

使用する組み込みライブラリの型定義を指定します。

```json
{
  "lib": ["ES2022", "DOM", "DOM.Iterable"]
  // Webアプリ: DOM, DOM.Iterable を含める
  // Node.js: DOM を除外
}
```

### outDir / rootDir

```json
{
  "outDir": "./dist",    // コンパイル結果の出力先
  "rootDir": "./src"     // ソースファイルのルート
}
```

### declaration / declarationMap

```json
{
  "declaration": true,      // .d.ts ファイルを生成
  "declarationMap": true    // .d.ts.map ファイルも生成
}
```

ライブラリを公開する場合に必要です。

### sourceMap

```json
{
  "sourceMap": true  // .js.map ファイルを生成
}
```

デバッグ時にTypeScriptのソースコードにマッピングできます。

## パス関連

### paths

```json
{
  "baseUrl": ".",
  "paths": {
    "@/*": ["./src/*"],
    "@components/*": ["./src/components/*"],
    "@utils/*": ["./src/utils/*"],
    "@types/*": ["./src/types/*"]
  }
}
```

**注意：** `paths` はTypeScriptの型チェック時のみ使用されます。実行時のパス解決には、バンドラーの設定（Viteの `resolve.alias` 等）も必要です。

## プロジェクト参照

大規模プロジェクトでは、プロジェクト参照を使って複数の `tsconfig.json` を管理します。

```json
// tsconfig.json（ルート）
{
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/ui" },
    { "path": "./packages/api" }
  ]
}
```

```json
// packages/core/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist"
  }
}
```

## フレームワーク別の推奨設定

### Next.js

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Node.js（ESM）

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "strict": true,
    "outDir": "./dist",
    "declaration": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

## まとめ

- `strict: true` は必ず有効にする（新規プロジェクト）
- `noUncheckedIndexedAccess` も追加で有効にすることを推奨
- `moduleResolution: "bundler"` がバンドラー使用時の推奨
- `isolatedModules: true` でトランスパイラとの互換性を確保
- `skipLibCheck: true` でビルド速度を改善
- フレームワークの推奨設定をベースにカスタマイズする

次回は「型安全なエラーハンドリング」について詳しく解説します。
