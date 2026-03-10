# 【SQL入門】第6回：INSERT, UPDATE, DELETE — データの追加・更新・削除

## はじめに

これまではSELECT文でデータを「読む」方法を学んできました。今回は**DML（Data Manipulation Language）**のうち、データの追加（INSERT）、更新（UPDATE）、削除（DELETE）を解説します。

## INSERT：データの追加

### 基本構文

```sql
INSERT INTO テーブル名 (カラム1, カラム2, ...)
VALUES (値1, 値2, ...);
```

### 全カラムを指定して追加

```sql
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES (9, '木村拓哉', 29, '営業部', 460000, '2023-10-01');
```

### カラム指定を省略する場合

テーブルの全カラムに対して順番通りに値を指定する場合、カラム名を省略できます。

```sql
INSERT INTO employees
VALUES (10, '松本潤', 33, '開発部', 550000, '2017-04-01');
```

> ⚠️ カラム名を省略すると、テーブル構造が変更された際にエラーの原因になります。**カラム名は明示的に指定する**ことを推奨します。

### 一部のカラムだけ指定

指定しなかったカラムにはデフォルト値またはNULLが入ります。

```sql
INSERT INTO employees (emp_id, name, age)
VALUES (11, '新入社員', 22);
-- department, salary, hire_date はNULL（またはデフォルト値）
```

### 複数行を一度に追加

```sql
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES
    (12, '吉田大輔', 26, '経理部', 420000, '2024-04-01'),
    (13, '小林真由', 31, '開発部', 510000, '2020-10-01'),
    (14, '加藤悠太', 24, '営業部', 400000, '2025-04-01');
```

### SELECT結果をINSERT

別のテーブルやクエリの結果を挿入できます。

```sql
-- 別テーブルのデータを挿入
INSERT INTO employees_backup (emp_id, name, age, department, salary, hire_date)
SELECT emp_id, name, age, department, salary, hire_date
FROM employees
WHERE department = '開発部';
```

### INSERT時のデフォルト値

```sql
-- DEFAULTキーワードでデフォルト値を明示的に使用
INSERT INTO employees (emp_id, name, age, department)
VALUES (15, '新人', 22, DEFAULT);
```

## UPDATE：データの更新

### 基本構文

```sql
UPDATE テーブル名
SET カラム1 = 新しい値1, カラム2 = 新しい値2
WHERE 条件;
```

> ⚠️ **重要**：**WHERE句を忘れると全行が更新されます**。UPDATE文は必ずWHERE句をつけることを習慣にしましょう。

### 特定の行を更新

```sql
-- emp_id=1の社員の給与を更新
UPDATE employees
SET salary = 480000
WHERE emp_id = 1;
```

### 複数カラムを同時に更新

```sql
-- emp_id=2の社員の部署と給与を更新
UPDATE employees
SET department = '営業部',
    salary = 530000
WHERE emp_id = 2;
```

### 計算式を使った更新

```sql
-- 全社員の給与を5%アップ
UPDATE employees
SET salary = salary * 1.05;

-- 開発部の社員のみ10%アップ
UPDATE employees
SET salary = salary * 1.10
WHERE department = '開発部';
```

### 条件付き更新（CASE式）

```sql
-- 部署に応じた昇給
UPDATE employees
SET salary = CASE
    WHEN department = '開発部' THEN salary * 1.10
    WHEN department = '営業部' THEN salary * 1.08
    ELSE salary * 1.05
END;
```

### 別テーブルの値で更新

```sql
-- PostgreSQL
UPDATE employees e
SET department = d.new_department
FROM department_changes d
WHERE e.emp_id = d.emp_id;

-- MySQL
UPDATE employees e
JOIN department_changes d ON e.emp_id = d.emp_id
SET e.department = d.new_department;
```

## DELETE：データの削除

### 基本構文

```sql
DELETE FROM テーブル名
WHERE 条件;
```

> ⚠️ **重要**：**WHERE句を忘れると全行が削除されます**。本番環境では特に注意してください。

### 特定の行を削除

```sql
-- emp_id=9の社員を削除
DELETE FROM employees
WHERE emp_id = 9;
```

### 条件に合う複数行を削除

```sql
-- 人事部の全社員を削除
DELETE FROM employees
WHERE department = '人事部';
```

### 全行を削除

```sql
-- DELETEで全行削除（ログが残り、ロールバック可能）
DELETE FROM employees;

-- TRUNCATEで全行削除（高速だが、ロールバックできない場合がある）
TRUNCATE TABLE employees;
```

| 比較 | DELETE | TRUNCATE |
|------|--------|----------|
| WHERE句 | 使える | 使えない |
| 速度 | 遅い（行ごとに処理） | 速い（一括処理） |
| ロールバック | 可能 | RDBMS依存 |
| トリガー | 発火する | 発火しない |
| AUTO_INCREMENT | リセットされない | リセットされる |

## 安全なデータ操作のための実践テクニック

### 1. 更新前に確認する

UPDATE/DELETEを実行する前に、同じWHERE句でSELECTを実行して影響範囲を確認しましょう。

```sql
-- ステップ1：影響範囲の確認
SELECT * FROM employees
WHERE department = '営業部' AND salary < 450000;

-- ステップ2：確認した上で更新
UPDATE employees
SET salary = salary * 1.05
WHERE department = '営業部' AND salary < 450000;
```

### 2. トランザクションを使う

```sql
-- トランザクション内で実行
BEGIN;

UPDATE employees
SET salary = salary * 1.10
WHERE department = '開発部';

-- 結果を確認
SELECT * FROM employees WHERE department = '開発部';

-- 問題なければ確定、問題があれば取消
COMMIT;    -- 確定
-- ROLLBACK;  -- 取消
```

### 3. LIMIT句で安全網を張る（MySQL）

```sql
-- MySQLではDELETEにLIMITを付けられる
DELETE FROM employees
WHERE department = '営業部'
LIMIT 10;
```

### 4. RETURNING句で結果を確認（PostgreSQL）

```sql
-- 更新された行を返す
UPDATE employees
SET salary = salary * 1.05
WHERE department = '開発部'
RETURNING emp_id, name, salary;

-- 削除された行を返す
DELETE FROM employees
WHERE emp_id = 9
RETURNING *;
```

## UPSERT：存在すれば更新、なければ挿入

### PostgreSQL（ON CONFLICT）

```sql
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES (1, '田中太郎', 31, '営業部', 500000, '2020-04-01')
ON CONFLICT (emp_id)
DO UPDATE SET
    age = EXCLUDED.age,
    salary = EXCLUDED.salary;
```

### MySQL（ON DUPLICATE KEY UPDATE）

```sql
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES (1, '田中太郎', 31, '営業部', 500000, '2020-04-01')
ON DUPLICATE KEY UPDATE
    age = VALUES(age),
    salary = VALUES(salary);
```

### SQLite（ON CONFLICT）

```sql
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES (1, '田中太郎', 31, '営業部', 500000, '2020-04-01')
ON CONFLICT(emp_id)
DO UPDATE SET
    age = excluded.age,
    salary = excluded.salary;
```

## MERGE文（SQL標準）

SQL標準の`MERGE`文を使うと、条件に応じたINSERT/UPDATE/DELETEを1つの文で行えます。

```sql
-- SQL Server / Oracle
MERGE INTO employees AS target
USING new_employees AS source
ON target.emp_id = source.emp_id
WHEN MATCHED THEN
    UPDATE SET
        target.salary = source.salary,
        target.department = source.department
WHEN NOT MATCHED THEN
    INSERT (emp_id, name, age, department, salary, hire_date)
    VALUES (source.emp_id, source.name, source.age,
            source.department, source.salary, source.hire_date);
```

## 実践練習

```sql
-- 練習1：新しい社員を追加
INSERT INTO employees (emp_id, name, age, department, salary, hire_date)
VALUES (20, '練習太郎', 25, '開発部', 430000, '2025-04-01');

-- 練習2：全社員の給与を3%アップ
BEGIN;
UPDATE employees SET salary = ROUND(salary * 1.03);
SELECT name, salary FROM employees;
COMMIT;

-- 練習3：特定の社員を削除（先に確認してから）
SELECT * FROM employees WHERE emp_id = 20;
DELETE FROM employees WHERE emp_id = 20;

-- 練習4：部署に応じた給与更新
UPDATE employees
SET salary = CASE
    WHEN department = '開発部' THEN salary + 20000
    WHEN department = '営業部' THEN salary + 15000
    ELSE salary + 10000
END;
```

## まとめ

| 操作 | 構文 | 注意点 |
|------|------|--------|
| 追加 | `INSERT INTO ... VALUES` | カラム名を明示的に指定する |
| 更新 | `UPDATE ... SET ... WHERE` | WHERE句を忘れない |
| 削除 | `DELETE FROM ... WHERE` | WHERE句を忘れない |
| 全削除 | `TRUNCATE TABLE` | ロールバック不可の場合あり |
| UPSERT | `ON CONFLICT` / `ON DUPLICATE KEY` | RDBMS依存 |

次回は、複数のテーブルを結合する**JOIN**の基本（INNER JOIN）を学びます。
