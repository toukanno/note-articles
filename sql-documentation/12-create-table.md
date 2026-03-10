# 【SQL入門】第12回：CREATE TABLE — テーブル設計の基本

## はじめに

これまでDML（データ操作言語）を中心に学んできました。今回からは**DDL（データ定義言語）**に進み、テーブルの作成と設計の基本を学びます。

## CREATE TABLEの基本構文

```sql
CREATE TABLE テーブル名 (
    カラム名1 データ型 [制約],
    カラム名2 データ型 [制約],
    ...
    [テーブル制約]
);
```

## データ型

### 数値型

| データ型 | 説明 | 範囲（目安） |
|---------|------|-------------|
| `SMALLINT` | 小さい整数 | -32,768 〜 32,767 |
| `INTEGER` / `INT` | 整数 | -21億 〜 21億 |
| `BIGINT` | 大きい整数 | -922京 〜 922京 |
| `DECIMAL(p,s)` | 固定小数点 | 精度p桁、小数s桁 |
| `NUMERIC(p,s)` | 固定小数点 | DECIMALと同等 |
| `REAL` | 浮動小数点（単精度） | 約7桁の精度 |
| `DOUBLE PRECISION` | 浮動小数点（倍精度） | 約15桁の精度 |

```sql
CREATE TABLE products (
    product_id INTEGER,
    price DECIMAL(10, 2),      -- 99,999,999.99まで
    weight DOUBLE PRECISION,
    stock_count SMALLINT
);
```

> 💡 **金額にはDECIMAL/NUMERICを使用**しましょう。REAL/DOUBLEは誤差が発生する可能性があります。

### 文字列型

| データ型 | 説明 | 特徴 |
|---------|------|------|
| `CHAR(n)` | 固定長文字列 | 常にn文字（余白はスペース埋め） |
| `VARCHAR(n)` | 可変長文字列 | 最大n文字 |
| `TEXT` | 可変長文字列（長さ制限なし） | 長い文章向け |

```sql
CREATE TABLE users (
    user_code CHAR(8),           -- 固定8文字のコード
    username VARCHAR(50),        -- 最大50文字
    bio TEXT                     -- 長さ無制限のプロフィール
);
```

### 日付・時刻型

| データ型 | 説明 | 例 |
|---------|------|-----|
| `DATE` | 日付 | `2024-01-15` |
| `TIME` | 時刻 | `14:30:00` |
| `TIMESTAMP` | 日付+時刻 | `2024-01-15 14:30:00` |
| `INTERVAL` | 期間 | `3 days`, `2 hours` |

```sql
CREATE TABLE events (
    event_id INTEGER,
    event_date DATE,
    start_time TIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 論理型

```sql
CREATE TABLE settings (
    setting_name VARCHAR(50),
    is_enabled BOOLEAN DEFAULT TRUE    -- TRUE / FALSE
);
```

> 📝 MySQLにはBOOLEAN型がありますが、内部的にはTINYINT(1)です。

### RDBMS固有の型

```sql
-- PostgreSQL
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,            -- 自動採番（PostgreSQL）
    data JSONB,                       -- JSONデータ
    tags TEXT[],                      -- 配列
    search_vector TSVECTOR            -- 全文検索
);

-- MySQL
CREATE TABLE documents (
    id INT AUTO_INCREMENT PRIMARY KEY, -- 自動採番（MySQL）
    data JSON,                         -- JSONデータ
    content MEDIUMTEXT                 -- 16MBまでのテキスト
);
```

## テーブル設計の例

### ECサイトの注文管理

```sql
-- 顧客テーブル
CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(200) UNIQUE NOT NULL,
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 商品テーブル
CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10, 2) NOT NULL,
    stock INTEGER DEFAULT 0,
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE
);

-- 注文テーブル
CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(12, 2),
    status VARCHAR(20) DEFAULT 'pending'
);

-- 注文明細テーブル
CREATE TABLE order_items (
    order_item_id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL
);
```

## ALTER TABLE：テーブル構造の変更

### カラムの追加

```sql
ALTER TABLE employees ADD COLUMN email VARCHAR(200);
ALTER TABLE employees ADD COLUMN phone VARCHAR(20) DEFAULT 'N/A';
```

### カラムの削除

```sql
ALTER TABLE employees DROP COLUMN phone;
```

### カラムの型変更

```sql
-- PostgreSQL
ALTER TABLE employees ALTER COLUMN name TYPE VARCHAR(200);

-- MySQL
ALTER TABLE employees MODIFY COLUMN name VARCHAR(200);
```

### カラム名の変更

```sql
-- PostgreSQL
ALTER TABLE employees RENAME COLUMN name TO full_name;

-- MySQL
ALTER TABLE employees CHANGE COLUMN name full_name VARCHAR(200);
```

### テーブル名の変更

```sql
ALTER TABLE employees RENAME TO staff;
-- または
RENAME TABLE employees TO staff;  -- MySQL
```

## DROP TABLE：テーブルの削除

```sql
-- テーブルを削除
DROP TABLE employees;

-- テーブルが存在する場合のみ削除（エラー回避）
DROP TABLE IF EXISTS employees;

-- 依存関係があるテーブルも強制削除（PostgreSQL）
DROP TABLE employees CASCADE;
```

> ⚠️ DROP TABLEは取り消せません。本番環境では十分注意してください。

## 一時テーブル

セッション終了時に自動的に削除されるテーブルです。

```sql
-- 一時テーブルの作成
CREATE TEMPORARY TABLE temp_results (
    name VARCHAR(100),
    score INTEGER
);

-- SELECT結果から一時テーブルを作成
CREATE TEMPORARY TABLE high_salary_emp AS
SELECT * FROM employees WHERE salary >= 500000;
```

## CREATE TABLE AS SELECT

既存のクエリ結果からテーブルを作成します。

```sql
-- クエリ結果をテーブル化
CREATE TABLE department_stats AS
SELECT
    department,
    COUNT(*) AS member_count,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```

## 命名規則のベストプラクティス

| ルール | 良い例 | 悪い例 |
|--------|--------|--------|
| テーブルは複数形 | `employees` | `employee` |
| スネークケース | `order_items` | `orderItems`, `OrderItems` |
| わかりやすい名前 | `customer_id` | `cid`, `id1` |
| 予約語を避ける | `order_date` | `order`, `date` |
| 略語を避ける | `department` | `dept` |
| 主キーは`テーブル名_id` | `employee_id` | `id` |

## 実践練習

```sql
-- 練習1：ブログシステムのテーブルを作成
CREATE TABLE authors (
    author_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(200) UNIQUE
);

CREATE TABLE posts (
    post_id INTEGER PRIMARY KEY,
    author_id INTEGER NOT NULL,
    title VARCHAR(300) NOT NULL,
    body TEXT,
    published_at TIMESTAMP,
    is_draft BOOLEAN DEFAULT TRUE
);

CREATE TABLE comments (
    comment_id INTEGER PRIMARY KEY,
    post_id INTEGER NOT NULL,
    commenter_name VARCHAR(100),
    body TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 練習2：既存テーブルにカラムを追加
ALTER TABLE posts ADD COLUMN view_count INTEGER DEFAULT 0;
ALTER TABLE posts ADD COLUMN category VARCHAR(50);

-- 練習3：クエリ結果からテーブルを作成
CREATE TABLE salary_report AS
SELECT department, AVG(salary) AS avg_salary, MAX(salary) AS max_salary
FROM employees
GROUP BY department;
```

## まとめ

| 操作 | 構文 |
|------|------|
| テーブル作成 | `CREATE TABLE t (col type)` |
| カラム追加 | `ALTER TABLE t ADD COLUMN col type` |
| カラム削除 | `ALTER TABLE t DROP COLUMN col` |
| 型変更 | `ALTER TABLE t ALTER COLUMN col TYPE type` |
| テーブル削除 | `DROP TABLE t` |
| テーブル複製 | `CREATE TABLE t AS SELECT ...` |

次回は、データの正確性を保つための**制約（NOT NULL, UNIQUE, CHECK, FOREIGN KEY）**を学びます。
