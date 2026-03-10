# 【SQL入門】第5回：集約関数 — COUNT, SUM, AVG, MAX, MIN

## はじめに

前回はORDER BYとLIMITを学びました。今回は、データの件数や合計、平均値などを計算する**集約関数（Aggregate Functions）**を解説します。

## 集約関数とは

集約関数は、複数の行の値をまとめて1つの結果を返す関数です。

| 関数 | 説明 |
|------|------|
| `COUNT()` | 行数をカウント |
| `SUM()` | 合計値 |
| `AVG()` | 平均値 |
| `MAX()` | 最大値 |
| `MIN()` | 最小値 |

## COUNT：行数をカウント

### COUNT(*)

テーブルの全行数をカウントします（NULLを含む）。

```sql
SELECT COUNT(*) FROM employees;
```

結果：

```
count
-----
    8
```

### COUNT(カラム名)

指定カラムがNULLでない行数をカウントします。

```sql
SELECT COUNT(department) FROM employees;
```

### COUNT(DISTINCT カラム名)

重複を排除した上でカウントします。

```sql
-- 部署の種類数
SELECT COUNT(DISTINCT department) FROM employees;
```

結果：

```
count
-----
    4
```

## SUM：合計値

```sql
-- 給与の合計
SELECT SUM(salary) AS total_salary FROM employees;
```

結果：

```
total_salary
-----------
    3840000
```

> ⚠️ SUMは数値型のカラムにのみ使用できます。NULLは無視されます。

## AVG：平均値

```sql
-- 給与の平均
SELECT AVG(salary) AS avg_salary FROM employees;
```

結果：

```
avg_salary
----------
  480000.0
```

### NULLに注意

AVGはNULLの行を**除外して計算**します。

```sql
-- 例：5行中2行がNULLの場合
-- 値: 100, 200, NULL, 300, NULL
-- AVG = (100 + 200 + 300) / 3 = 200（5で割らない）
```

NULLを0として計算したい場合：

```sql
SELECT AVG(COALESCE(salary, 0)) AS avg_salary FROM employees;
```

## MAX / MIN：最大値・最小値

```sql
-- 最高給与と最低給与
SELECT
    MAX(salary) AS max_salary,
    MIN(salary) AS min_salary
FROM employees;
```

結果：

```
max_salary | min_salary
-----------+-----------
    600000 |     380000
```

### 日付や文字列にも使える

```sql
-- 最も古い入社日と最も新しい入社日
SELECT
    MIN(hire_date) AS earliest,
    MAX(hire_date) AS latest
FROM employees;
```

結果：

```
earliest   | latest
-----------+-----------
2015-04-01 | 2024-04-01
```

## 複数の集約関数を同時に使う

```sql
SELECT
    COUNT(*) AS 社員数,
    SUM(salary) AS 給与合計,
    AVG(salary) AS 給与平均,
    MAX(salary) AS 最高給与,
    MIN(salary) AS 最低給与,
    MAX(salary) - MIN(salary) AS 給与幅
FROM employees;
```

結果：

```
社員数 | 給与合計 | 給与平均 | 最高給与 | 最低給与 | 給与幅
------+---------+---------+---------+---------+-------
    8 | 3840000 | 480000  |  600000 |  380000 | 220000
```

## WHERE句との組み合わせ

集約関数はWHERE句で絞り込んだ後のデータに対して計算されます。

```sql
-- 開発部の社員数と平均給与
SELECT
    COUNT(*) AS 開発部員数,
    AVG(salary) AS 平均給与
FROM employees
WHERE department = '開発部';
```

結果：

```
開発部員数 | 平均給与
----------+-----------
        3 | 540000.0
```

```sql
-- 2020年以降に入社した社員の統計
SELECT
    COUNT(*) AS 社員数,
    AVG(age) AS 平均年齢,
    AVG(salary) AS 平均給与
FROM employees
WHERE hire_date >= '2020-01-01';
```

## 集約関数の注意点

### 1. 集約関数とカラムの混在

集約関数を使わないカラムと集約関数を同じSELECT句に入れることは基本的にできません。

```sql
-- ❌ エラーになる（MySQL以外）
SELECT name, COUNT(*) FROM employees;

-- ✅ GROUP BYを使う（次回詳しく解説）
SELECT department, COUNT(*) FROM employees
GROUP BY department;
```

### 2. WHERE句で集約関数は使えない

```sql
-- ❌ エラーになる
SELECT * FROM employees
WHERE salary > AVG(salary);

-- ✅ サブクエリを使う
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

### 3. NULLの扱い

| 関数 | NULLの扱い |
|------|-----------|
| `COUNT(*)` | NULLを含めてカウント |
| `COUNT(col)` | NULLを除外してカウント |
| `SUM(col)` | NULLを無視 |
| `AVG(col)` | NULLを無視（分母にも含まない） |
| `MAX(col)` | NULLを無視 |
| `MIN(col)` | NULLを無視 |

## 条件付きカウント

CASE式と組み合わせて、条件付きのカウントや合計を行えます。

```sql
SELECT
    COUNT(*) AS 全社員数,
    COUNT(CASE WHEN department = '開発部' THEN 1 END) AS 開発部人数,
    COUNT(CASE WHEN department = '営業部' THEN 1 END) AS 営業部人数,
    COUNT(CASE WHEN salary >= 500000 THEN 1 END) AS 高給与人数
FROM employees;
```

結果：

```
全社員数 | 開発部人数 | 営業部人数 | 高給与人数
--------+----------+----------+----------
      8 |        3 |        2 |        3
```

### FILTER句（PostgreSQL）

PostgreSQLでは、より読みやすい`FILTER`構文が使えます。

```sql
SELECT
    COUNT(*) AS 全社員数,
    COUNT(*) FILTER (WHERE department = '開発部') AS 開発部人数,
    AVG(salary) FILTER (WHERE department = '開発部') AS 開発部平均給与
FROM employees;
```

## 統計的な関数

多くのRDBMSでは、標準偏差や分散を計算する関数も提供されています。

```sql
-- 標準偏差（Standard Deviation）
SELECT
    STDDEV(salary) AS salary_stddev,      -- 標本標準偏差
    STDDEV_POP(salary) AS salary_stddev_pop  -- 母集団標準偏差
FROM employees;

-- 分散（Variance）
SELECT
    VARIANCE(salary) AS salary_var,
    VAR_POP(salary) AS salary_var_pop
FROM employees;
```

## 実践練習

```sql
-- 練習1：全社員の平均年齢を求める
SELECT AVG(age) AS 平均年齢 FROM employees;

-- 練習2：部署ごとの社員数を求める（GROUP BYの予習）
SELECT department, COUNT(*) AS 人数
FROM employees
GROUP BY department;

-- 練習3：2020年以降入社の社員の最高給与と最低給与の差
SELECT MAX(salary) - MIN(salary) AS 給与差
FROM employees
WHERE hire_date >= '2020-01-01';

-- 練習4：給与50万円以上の社員数と全社員数を同時に取得
SELECT
    COUNT(*) AS 全社員数,
    COUNT(CASE WHEN salary >= 500000 THEN 1 END) AS 高給与社員数
FROM employees;
```

## まとめ

| 関数 | 用途 | NULL | 例 |
|------|------|------|----|
| `COUNT(*)` | 全行数 | 含む | `COUNT(*)` → 8 |
| `COUNT(col)` | 非NULL行数 | 除外 | `COUNT(dept)` |
| `SUM(col)` | 合計 | 無視 | `SUM(salary)` |
| `AVG(col)` | 平均 | 無視 | `AVG(salary)` |
| `MAX(col)` | 最大値 | 無視 | `MAX(salary)` |
| `MIN(col)` | 最小値 | 無視 | `MIN(hire_date)` |

次回は、INSERT・UPDATE・DELETEを使ったデータの追加・更新・削除を学びます。
