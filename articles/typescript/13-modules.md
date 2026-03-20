# TypeScriptのモジュールシステムを理解する

## はじめに

モジュールシステムは、コードを分割し再利用可能にするための仕組みです。TypeScriptはESモジュール（ESM）をベースとしたモジュールシステムを採用しており、CommonJS（CJS）との相互運用もサポートしています。

## ESモジュールの基本

### 名前付きエクスポート

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export const PI = 3.14159;
```

### 名前付きインポート

```typescript
// app.ts
import { add, subtract, PI } from "./math";

console.log(add(1, 2));
console.log(PI);
```

### エイリアスを使ったインポート

```typescript
import { add as sum, subtract as minus } from "./math";

console.log(sum(1, 2));
```

### デフォルトエクスポート

```typescript
// logger.ts
export default class Logger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}
```

```typescript
// app.ts
import Logger from "./logger";
const logger = new Logger();
```

### 名前空間インポート

```typescript
import * as MathUtils from "./math";

console.log(MathUtils.add(1, 2));
console.log(MathUtils.PI);
```

### 再エクスポート

```typescript
// index.ts（バレルファイル）
export { add, subtract } from "./math";
export { default as Logger } from "./logger";
export type { User } from "./types";
```

## 型のエクスポート・インポート

### type-only エクスポート/インポート

型だけをインポート/エクスポートする場合、`type` キーワードを使います。

```typescript
// types.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

export type Status = "active" | "inactive";
```

```typescript
// app.ts
import type { User, Status } from "./types";

// type-only インポートはランタイムに残らない
// import { User } from "./types" でも動くが、
// import type の方が意図が明確

const user: User = { id: 1, name: "太郎", email: "taro@example.com" };
```

### インラインtype修飾子

```typescript
// 値と型を混ぜてインポートする場合
import { createUser, type User, type CreateUserInput } from "./user";

// createUser は値として使う
// User と CreateUserInput は型としてのみ使う
const input: CreateUserInput = { name: "太郎", email: "taro@example.com" };
const user: User = createUser(input);
```

## モジュール解決

### パスエイリアス

`tsconfig.json` で パスエイリアスを設定できます。

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

```typescript
// エイリアスを使ったインポート
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";
```

### Node.js モジュール解決

```json
{
  "compilerOptions": {
    "moduleResolution": "node"    // Node.js 方式
    // または
    // "moduleResolution": "bundler" // バンドラー使用時（推奨）
  }
}
```

## CommonJS との相互運用

### ESMからCJSモジュールをインポート

```typescript
// CommonJS モジュールのインポート
import express from "express";  // デフォルトインポートとして扱う

// esModuleInterop が有効な場合（推奨）
import path from "path";
import fs from "fs";
```

### tsconfig.json の設定

```json
{
  "compilerOptions": {
    "esModuleInterop": true,     // CJSモジュールのデフォルトインポートを許可
    "allowSyntheticDefaultImports": true
  }
}
```

## 動的インポート

```typescript
// 動的インポート（コード分割）
async function loadModule() {
  const { add } = await import("./math");
  console.log(add(1, 2));
}

// 条件付きインポート
async function loadLocale(lang: string) {
  const translations = await import(`./locales/${lang}.json`);
  return translations.default;
}
```

## アンビエント宣言（declare）

型情報のないJavaScriptモジュールに型を付与する方法です。

### グローバル宣言

```typescript
// global.d.ts
declare global {
  interface Window {
    analytics: {
      track(event: string, data?: Record<string, unknown>): void;
    };
  }
}

export {}; // ファイルをモジュールとして扱うため
```

### モジュール宣言

```typescript
// declarations.d.ts

// 型定義のないモジュールに型を付与
declare module "untyped-library" {
  export function doSomething(input: string): number;
}

// ワイルドカードモジュール宣言
declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}

declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const src: string;
  export default src;
}
```

## バレルファイル（index.ts）

ディレクトリ内のモジュールを1つのエントリポイントからエクスポートするパターンです。

```
src/
  components/
    Button.tsx
    Input.tsx
    Modal.tsx
    index.ts      ← バレルファイル
```

```typescript
// components/index.ts
export { Button } from "./Button";
export { Input } from "./Input";
export { Modal } from "./Modal";
```

```typescript
// 使用側
import { Button, Input, Modal } from "./components";
```

### バレルファイルの注意点

- バンドルサイズが大きくなる可能性がある（Tree-shakingが効かない場合）
- 循環参照の原因になることがある
- ビルド時間が増加することがある
- 大規模プロジェクトでは慎重に使うべき

## モジュールの設計パターン

### 機能ごとのモジュール構成

```
src/
  features/
    auth/
      types.ts
      api.ts
      hooks.ts
      components/
      index.ts
    users/
      types.ts
      api.ts
      hooks.ts
      components/
      index.ts
```

### レイヤー型のモジュール構成

```
src/
  domain/        ← ビジネスロジック・型定義
  application/   ← ユースケース
  infrastructure/ ← 外部サービス連携
  presentation/  ← UI
```

## まとめ

- TypeScriptはESモジュールベースのモジュールシステムを採用
- `import type` で型だけをインポートし、ランタイムコードを減らせる
- パスエイリアスでインポートパスを短縮できる
- `esModuleInterop` でCommonJSモジュールとの相互運用が容易に
- アンビエント宣言で型定義のないモジュールにも型を付与できる
- バレルファイルは便利だが、パフォーマンスへの影響に注意

次回は「デコレータ入門」について詳しく解説します。
