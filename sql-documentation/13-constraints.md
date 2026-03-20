# 【SQL入門】第13回：制約 — NOT NULL, UNIQUE, CHECK, FOREIGN KEY

## はじめに

前回はCREATE TABLEの基本を学びました。今回は、データの正確性と整合性を保つための**制約（Constraint）**を詳しく解説します。

## 制約の種類

| 制約 | 説明 |
|------|------|
| `NOT NULL` | NULL値を禁止する |
| `UNIQUE` | 重複値を禁止する |
| `PRIMARY KEY` | 主キー（NOT NULL + UNIQUE） |
| `FOREIGN KEY` | 外部キー（他テーブルとの参照整合性） |
| `CHECK` | 任意の条件を満たすことを要求する |
| `DEFAULT` | デフォルト値を設定する |

## NOT NULL：NULLを禁止

```sql
CREATE TABLE employees (
    emp_id INTEGER NOT NULL,
    name VARCHAR(100) NOT NULL,     -- 名前は必須
    email VARCHAR(200),             -- メールは任意（NULL許可）
    department VARCHAR(50) NOT NULL  -- 部署は必須
);
```

```sql
-- ✅ 成功
INSERT INTO employees (emp_id, name, department)
VALUES (1, '田中太郎', '営業部');

-- ❌ エラー：nameがNULL
INSERT INTO employees (emp_id, name, department)
VALUES (2, NULL, '開発部');
```

## UNIQUE：重複を禁止

```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    username VARCHAR(50) UNIQUE,     -- ユーザー名は一意
    email VARCHAR(200) UNIQUE        -- メールアドレスも一意
);
```

```sql
INSERT INTO users VALUES (1, 'tanaka', 'tanaka@example.com');
INSERT INTO users VALUES (2, 'suzuki', 'suzuki@example.com');

-- ❌ エラー：usernameが重複
INSERT INTO users VALUES (3, 'tanaka', 'another@example.com');
```

### 複合UNIQUE制約

複数カラムの組み合わせで一意性を保証します。

```sql
CREATE TABLE course_enrollment (
    student_id INTEGER,
    course_id INTEGER,
    enrolled_at TIMESTAMP,
    UNIQUE (student_id, course_id)   -- 同じ学生が同じ講座に2回登録できない
);
```

> 📝 UNIQUE制約ではNULLは重複とみなされません（RDBMSによる差異あり）。

## PRIMARY KEY：主キー

各レコードを一意に識別するカラムです。`NOT NULL + UNIQUE`の効果があります。

```sql
-- カラム制約として定義
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- テーブル制約として定義
CREATE TABLE employees (
    emp_id INTEGER,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (emp_id)
);
```

### 複合主キー

```sql
CREATE TABLE order_items (
    order_id INTEGER,
    item_number INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    PRIMARY KEY (order_id, item_number)  -- 複合主キー
);
```

### 自動採番

```sql
-- PostgreSQL
CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,   -- 自動採番
    name VARCHAR(100) NOT NULL
);
-- PostgreSQL 10以降の推奨方法
CREATE TABLE employees (
    emp_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- MySQL
CREATE TABLE employees (
    emp_id INTEGER AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- SQLite
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(100) NOT NULL
);
```

## FOREIGN KEY：外部キー

他のテーブルの主キーを参照し、参照整合性を保証します。

```sql
CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL
);

CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    dept_id INTEGER,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

### 参照整合性の動作

```sql
-- ✅ 存在する部署IDなら挿入可能
INSERT INTO departments VALUES (1, '営業部');
INSERT INTO employees VALUES (1, '田中太郎', 1);

-- ❌ エラー：dept_id=99は departments に存在しない
INSERT INTO employees VALUES (2, '鈴木花子', 99);

-- ❌ エラー：dept_id=1は employees から参照されている
DELETE FROM departments WHERE dept_id = 1;
```

### ON DELETE / ON UPDATE（参照アクション）

親テーブルのデータが変更・削除された時の動作を指定できます。

```sql
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    dept_id INTEGER,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
        ON DELETE SET NULL       -- 親が削除されたらNULLにする
        ON UPDATE CASCADE        -- 親が更新されたら連動して更新
);
```

| アクション | 説明 |
|-----------|------|
| `CASCADE` | 親の変更/削除に連動 |
| `SET NULL` | NULLに設定 |
| `SET DEFAULT` | デフォルト値に設定 |
| `RESTRICT` | 親の変更/削除を禁止（デフォルト） |
| `NO ACTION` | RESTRICTと同様（チェックタイミングが異なる場合あり） |

```sql
-- CASCADE例：部署を削除すると、その部署の社員も削除される
FOREIGN KEY (dept_id) REFERENCES departments(dept_id) ON DELETE CASCADE

-- SET NULL例：部署を削除すると、社員のdept_idがNULLになる
FOREIGN KEY (dept_id) REFERENCES departments(dept_id) ON DELETE SET NULL

-- RESTRICT例：社員が存在する部署は削除できない（デフォルト）
FOREIGN KEY (dept_id) REFERENCES departments(dept_id) ON DELETE RESTRICT
```

## CHECK：値の制約

任意の条件式でデータを検証します。

```sql
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INTEGER CHECK (age >= 18 AND age <= 65),
    salary INTEGER CHECK (salary > 0),
    email VARCHAR(200) CHECK (email LIKE '%@%')
);
```

```sql
-- ✅ 成功
INSERT INTO employees VALUES (1, '田中太郎', 30, 450000, 'tanaka@example.com');

-- ❌ エラー：age < 18
INSERT INTO employees VALUES (2, '新人', 16, 300000, 'new@example.com');

-- ❌ エラー：salary <= 0
INSERT INTO employees VALUES (3, '山田', 25, -100, 'yamada@example.com');
```

### 名前付きCHECK制約

```sql
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INTEGER,
    salary INTEGER,
    CONSTRAINT chk_age CHECK (age >= 18 AND age <= 65),
    CONSTRAINT chk_salary CHECK (salary > 0)
);
```

名前付きにすると、エラーメッセージが分かりやすくなり、後から制約を削除しやすくなります。

## DEFAULT：デフォルト値

```sql
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    salary INTEGER DEFAULT 300000,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_admin BOOLEAN DEFAULT FALSE
);
```

```sql
-- DEFAULTが適用される
INSERT INTO employees (emp_id, name) VALUES (1, '田中太郎');
-- status='active', salary=300000, created_at=現在時刻, is_admin=FALSE
```

## 制約の追加・削除

### 制約の追加

```sql
-- NOT NULLの追加（PostgreSQL）
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;

-- UNIQUE制約の追加
ALTER TABLE employees ADD CONSTRAINT uq_email UNIQUE (email);

-- FOREIGN KEY制約の追加
ALTER TABLE employees
ADD CONSTRAINT fk_dept
FOREIGN KEY (dept_id) REFERENCES departments(dept_id);

-- CHECK制約の追加
ALTER TABLE employees
ADD CONSTRAINT chk_salary CHECK (salary > 0);
```

### 制約の削除

```sql
-- 名前付き制約の削除
ALTER TABLE employees DROP CONSTRAINT uq_email;
ALTER TABLE employees DROP CONSTRAINT fk_dept;

-- NOT NULLの削除（PostgreSQL）
ALTER TABLE employees ALTER COLUMN email DROP NOT NULL;
```

## 実践的なテーブル設計例

```sql
-- ECサイトのテーブル設計（制約付き）
CREATE TABLE customers (
    customer_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(200) NOT NULL UNIQUE,
    phone VARCHAR(20),
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'suspended')),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    product_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE orders (
    order_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_item_id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    UNIQUE (order_id, product_id)
);
```

## まとめ

| 制約 | 用途 | 例 |
|------|------|-----|
| NOT NULL | NULL禁止 | `name VARCHAR(100) NOT NULL` |
| UNIQUE | 重複禁止 | `email VARCHAR(200) UNIQUE` |
| PRIMARY KEY | 主キー | `emp_id INT PRIMARY KEY` |
| FOREIGN KEY | 参照整合性 | `REFERENCES departments(dept_id)` |
| CHECK | 値の検証 | `CHECK (age >= 18)` |
| DEFAULT | 既定値 | `DEFAULT CURRENT_TIMESTAMP` |

次回は、検索を高速化する**インデックス**の仕組みを学びます。
