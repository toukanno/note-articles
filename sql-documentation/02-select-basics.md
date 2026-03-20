# 【SQL入門】第2回：SELECT文の基本 — 必要なデータを取り出そう

## はじめに

前回はSQLとデータベースの基礎概念を学びました。今回はSQLで最も頻繁に使う**SELECT文**を深掘りし、テーブルから必要なデータだけを取り出す方法を学びます。

## SELECT文の基本構文

```sql
SELECT カラム名1, カラム名2, ...
FROM テーブル名;
```

## 全カラムの取得

アスタリスク（`*`）を使うと、テーブルのすべてのカラムを取得できます。

```sql
SELECT * FROM employees;
```

結果：

```
emp_id | name       | age | department | salary | hire_date
-------+------------+-----+------------+--------+-----------
     1 | 田中太郎   |  30 | 営業部     | 450000 | 2020-04-01
     2 | 鈴木花子   |  25 | 開発部     | 500000 | 2022-04-01
     3 | 佐藤次郎   |  35 | 人事部     | 480000 | 2018-04-01
     4 | 山田美咲   |  28 | 開発部     | 520000 | 2021-04-01
     5 | 高橋一郎   |  32 | 経理部     | 470000 | 2019-04-01
     6 | 伊藤由美   |  27 | 営業部     | 440000 | 2023-04-01
     7 | 渡辺健太   |  40 | 開発部     | 600000 | 2015-04-01
     8 | 中村さくら |  23 | 人事部     | 380000 | 2024-04-01
```

> ⚠️ **注意**：本番環境では `SELECT *` は避けましょう。必要なカラムだけを指定することで、パフォーマンスが向上し、コードの可読性も上がります。

## 特定のカラムを取得

カンマ区切りで必要なカラムを指定します。

```sql
SELECT name, department FROM employees;
```

結果：

```
name       | department
-----------+-----------
田中太郎   | 営業部
鈴木花子   | 開発部
佐藤次郎   | 人事部
山田美咲   | 開発部
高橋一郎   | 経理部
伊藤由美   | 営業部
渡辺健太   | 開発部
中村さくら | 人事部
```

## カラムに別名をつける（AS句）

`AS`を使って、出力結果のカラム名を変更できます。

```sql
SELECT
    name AS 社員名,
    department AS 部署,
    salary AS 月給
FROM employees;
```

結果：

```
社員名     | 部署   | 月給
-----------+--------+-------
田中太郎   | 営業部 | 450000
鈴木花子   | 開発部 | 500000
佐藤次郎   | 人事部 | 480000
...
```

`AS`は省略可能です：

```sql
-- 以下も同じ結果
SELECT name 社員名, department 部署 FROM employees;
```

> 💡 **ポイント**：別名にスペースや日本語を含む場合、ダブルクォートで囲むのが安全です。
> 例：`SELECT name AS "社員 名前" FROM employees;`

## 計算式を使う

SELECT文の中で計算を行うことができます。

```sql
-- 年収を計算
SELECT
    name AS 社員名,
    salary AS 月給,
    salary * 12 AS 年収
FROM employees;
```

結果：

```
社員名     | 月給   | 年収
-----------+--------+---------
田中太郎   | 450000 | 5400000
鈴木花子   | 500000 | 6000000
佐藤次郎   | 480000 | 5760000
山田美咲   | 520000 | 6240000
...
```

使える演算子：

| 演算子 | 意味 | 例 |
|--------|------|-----|
| `+` | 加算 | `salary + 50000` |
| `-` | 減算 | `salary - tax` |
| `*` | 乗算 | `salary * 12` |
| `/` | 除算 | `salary / 30` |
| `%` | 剰余 | `age % 10` |

## 文字列の結合

RDBMS によって文字列結合の方法が異なります。

```sql
-- PostgreSQL, SQLite（||演算子）
SELECT name || ' - ' || department AS 社員情報
FROM employees;

-- MySQL（CONCAT関数）
SELECT CONCAT(name, ' - ', department) AS 社員情報
FROM employees;

-- SQL Server（+演算子）
SELECT name + ' - ' + department AS 社員情報
FROM employees;
```

結果：

```
社員情報
-------------------
田中太郎 - 営業部
鈴木花子 - 開発部
佐藤次郎 - 人事部
...
```

## DISTINCT：重複を排除する

`DISTINCT`を使うと、重複した値を1つにまとめて表示できます。

```sql
-- 部署の一覧を取得（重複あり）
SELECT department FROM employees;
```

結果：

```
department
----------
営業部
開発部
人事部
開発部    ← 重複
経理部
営業部    ← 重複
開発部    ← 重複
人事部    ← 重複
```

```sql
-- 重複を排除
SELECT DISTINCT department FROM employees;
```

結果：

```
department
----------
営業部
開発部
人事部
経理部
```

### 複数カラムでのDISTINCT

```sql
-- 部署と年齢の組み合わせで重複を排除
SELECT DISTINCT department, age FROM employees;
```

この場合、`department`と`age`の**組み合わせ**が重複するレコードが排除されます。

## テーブルを使わないSELECT

テーブルからデータを取得せず、計算結果や固定値だけを取得することもできます。

```sql
-- 計算結果を取得
SELECT 1 + 1;

-- 現在の日付を取得（PostgreSQL）
SELECT CURRENT_DATE;

-- 現在の日時を取得（MySQL）
SELECT NOW();

-- 文字列を返す
SELECT 'Hello, SQL!' AS greeting;
```

> 📝 **補足**：Oracle Databaseでは`FROM DUAL`が必要です。
> `SELECT 1 + 1 FROM DUAL;`

## NULLの扱い

NULLは「値が存在しない」ことを表す特別な値です。SQLではNULLの扱いに注意が必要です。

```sql
-- NULLとの演算結果はNULL
SELECT 100 + NULL;  -- 結果: NULL
SELECT NULL * 5;     -- 結果: NULL
```

### NULLの判定

```sql
-- NULLの判定には IS NULL / IS NOT NULL を使う
SELECT * FROM employees WHERE department IS NULL;
SELECT * FROM employees WHERE department IS NOT NULL;

-- ❌ これは正しく動かない
SELECT * FROM employees WHERE department = NULL;
```

### COALESCE：NULLの置換

```sql
-- NULLの場合にデフォルト値を返す
SELECT
    name,
    COALESCE(department, '未配属') AS department
FROM employees;
```

## CASE式：条件分岐

SELECT文の中で条件分岐を行えます。

```sql
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 550000 THEN 'A'
        WHEN salary >= 450000 THEN 'B'
        WHEN salary >= 400000 THEN 'C'
        ELSE 'D'
    END AS salary_rank
FROM employees;
```

結果：

```
name       | salary | salary_rank
-----------+--------+------------
田中太郎   | 450000 | B
鈴木花子   | 500000 | B
佐藤次郎   | 480000 | B
山田美咲   | 520000 | B
高橋一郎   | 470000 | B
伊藤由美   | 440000 | C
渡辺健太   | 600000 | A
中村さくら | 380000 | D
```

### 単純CASE式

```sql
SELECT
    name,
    department,
    CASE department
        WHEN '開発部' THEN 'Engineering'
        WHEN '営業部' THEN 'Sales'
        WHEN '人事部' THEN 'HR'
        WHEN '経理部' THEN 'Accounting'
        ELSE 'Other'
    END AS dept_en
FROM employees;
```

## 型変換（CAST）

データ型を変換する場合に使います。

```sql
-- 数値を文字列に変換
SELECT CAST(salary AS VARCHAR(10)) FROM employees;

-- 文字列を数値に変換
SELECT CAST('12345' AS INTEGER);

-- 文字列を日付に変換
SELECT CAST('2024-01-15' AS DATE);
```

## よくあるミスと対処法

### 1. カンマの付け忘れ・余分なカンマ

```sql
-- ❌ カンマがない
SELECT name department FROM employees;

-- ❌ 末尾に余分なカンマ
SELECT name, department, FROM employees;

-- ✅ 正しい
SELECT name, department FROM employees;
```

### 2. 全角スペースの混入

日本語環境では全角スペースが混入しやすいので注意してください。SQLのキーワードやカラム名の間には半角スペースを使います。

### 3. 予約語との衝突

```sql
-- ❌ orderは予約語
SELECT order FROM orders;

-- ✅ ダブルクォートで囲む
SELECT "order" FROM orders;
```

## 実践練習

以下のSQLを実行して結果を確認してみましょう。

```sql
-- 練習1：社員名と年齢を取得
SELECT name, age FROM employees;

-- 練習2：年収（月給×12）と社員名を表示
SELECT name, salary * 12 AS annual_salary FROM employees;

-- 練習3：所属部署の一覧を取得（重複なし）
SELECT DISTINCT department FROM employees;

-- 練習4：給与ランクをCASE式で表示
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 500000 THEN '高'
        WHEN salary >= 400000 THEN '中'
        ELSE '低'
    END AS 給与レベル
FROM employees;
```

## まとめ

| 機能 | 構文 | 例 |
|------|------|-----|
| 全カラム取得 | `SELECT *` | `SELECT * FROM emp` |
| 特定カラム | `SELECT col1, col2` | `SELECT name, age FROM emp` |
| 別名 | `AS` | `SELECT name AS 名前` |
| 計算 | 演算子 | `SELECT salary * 12` |
| 重複排除 | `DISTINCT` | `SELECT DISTINCT dept` |
| NULL置換 | `COALESCE` | `COALESCE(col, '既定')` |
| 条件分岐 | `CASE WHEN` | `CASE WHEN x > 0 THEN...` |
| 型変換 | `CAST` | `CAST(col AS INT)` |

次回は`WHERE`句を使って、条件にマッチするデータだけを取り出す方法を学びます。
