# 【SQL入門】第4回：ORDER BYとLIMIT — データの並び替えと件数制限

## はじめに

前回はWHERE句で条件を指定してデータを絞り込む方法を学びました。今回は**ORDER BY**でデータを並び替え、**LIMIT**で取得件数を制限する方法を解説します。

## ORDER BY句の基本

```sql
SELECT カラム名
FROM テーブル名
ORDER BY カラム名 [ASC | DESC];
```

- `ASC`（Ascending）：昇順（小さい→大きい）。デフォルト
- `DESC`（Descending）：降順（大きい→小さい）

## 昇順ソート（ASC）

```sql
-- 年齢の昇順で並べる
SELECT name, age FROM employees
ORDER BY age ASC;
```

結果：

```
name       | age
-----------+----
中村さくら |  23
鈴木花子   |  25
伊藤由美   |  27
山田美咲   |  28
田中太郎   |  30
高橋一郎   |  32
佐藤次郎   |  35
渡辺健太   |  40
```

`ASC`は省略可能です：

```sql
-- 以下も同じ（ASCがデフォルト）
SELECT name, age FROM employees
ORDER BY age;
```

## 降順ソート（DESC）

```sql
-- 給与の高い順に並べる
SELECT name, salary FROM employees
ORDER BY salary DESC;
```

結果：

```
name       | salary
-----------+-------
渡辺健太   | 600000
山田美咲   | 520000
鈴木花子   | 500000
佐藤次郎   | 480000
高橋一郎   | 470000
田中太郎   | 450000
伊藤由美   | 440000
中村さくら | 380000
```

## 複数カラムでのソート

```sql
-- 部署名の昇順で並べ、同じ部署内では給与の降順で並べる
SELECT name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;
```

結果：

```
name       | department | salary
-----------+------------+-------
渡辺健太   | 開発部     | 600000
山田美咲   | 開発部     | 520000
鈴木花子   | 開発部     | 500000
経理部     | 経理部     | 470000
田中太郎   | 営業部     | 450000
伊藤由美   | 営業部     | 440000
佐藤次郎   | 人事部     | 480000
中村さくら | 人事部     | 380000
```

## カラム番号でソート

SELECT句で指定したカラムの位置番号でもソートできます。

```sql
-- 2番目のカラム（age）で昇順ソート
SELECT name, age, salary FROM employees
ORDER BY 2 ASC;
```

> ⚠️ カラム番号でのソートは可読性が低いため、**カラム名を使うことを推奨**します。特にSELECT句が変更された場合にバグの原因になります。

## 式や関数でソート

```sql
-- 年収（salary * 12）の降順で並べる
SELECT name, salary, salary * 12 AS annual_salary
FROM employees
ORDER BY salary * 12 DESC;

-- 別名でもソート可能（RDBMSによる）
SELECT name, salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;
```

## 日本語のソート

日本語のソート順は、データベースの**照合順序（Collation）**に依存します。

```sql
-- PostgreSQLで日本語の五十音順にソートする例
SELECT name FROM employees
ORDER BY name COLLATE "ja_JP.utf8";
```

## NULLのソート順

NULLを含むカラムをソートした場合、NULLの位置はRDBMSによって異なります。

| RDBMS | ASCでのNULLの位置 | DESCでのNULLの位置 |
|-------|-------------------|-------------------|
| PostgreSQL | 最後 | 最初 |
| MySQL | 最初 | 最後 |
| Oracle | 最後 | 最初 |
| SQL Server | 最初 | 最後 |

### NULLS FIRST / NULLS LAST

PostgreSQLやOracleでは、NULLの位置を明示的に指定できます。

```sql
-- NULLを最初に表示
SELECT name, department FROM employees
ORDER BY department ASC NULLS FIRST;

-- NULLを最後に表示
SELECT name, department FROM employees
ORDER BY department ASC NULLS LAST;
```

MySQLでは以下のテクニックが使えます：

```sql
-- MySQLでNULLを最後にする
SELECT name, department FROM employees
ORDER BY department IS NULL ASC, department ASC;
```

## LIMIT句：取得件数を制限する

### 基本構文

```sql
-- 先頭からN件だけ取得
SELECT カラム名
FROM テーブル名
LIMIT N;
```

```sql
-- 給与上位3名を取得
SELECT name, salary FROM employees
ORDER BY salary DESC
LIMIT 3;
```

結果：

```
name     | salary
---------+-------
渡辺健太 | 600000
山田美咲 | 520000
鈴木花子 | 500000
```

### OFFSET：スキップする行数を指定

```sql
-- 4番目から3件取得（上位3名を飛ばす）
SELECT name, salary FROM employees
ORDER BY salary DESC
LIMIT 3 OFFSET 3;
```

結果：

```
name     | salary
---------+-------
佐藤次郎 | 480000
高橋一郎 | 470000
田中太郎 | 450000
```

### ページネーションへの応用

```sql
-- 1ページあたり3件表示する場合
-- 1ページ目
SELECT * FROM employees ORDER BY emp_id LIMIT 3 OFFSET 0;

-- 2ページ目
SELECT * FROM employees ORDER BY emp_id LIMIT 3 OFFSET 3;

-- 3ページ目
SELECT * FROM employees ORDER BY emp_id LIMIT 3 OFFSET 6;

-- 一般式：N ページ目 → OFFSET (N - 1) * ページサイズ
```

> ⚠️ **注意**：OFFSETが大きくなるとパフォーマンスが低下します。大規模データのページネーションには**カーソルベース**の方法を検討してください。

```sql
-- カーソルベースのページネーション例
-- 前ページの最後のemp_idが5の場合
SELECT * FROM employees
WHERE emp_id > 5
ORDER BY emp_id
LIMIT 3;
```

## RDBMS別の構文の違い

### MySQL / PostgreSQL / SQLite

```sql
SELECT * FROM employees LIMIT 10 OFFSET 20;
-- または
SELECT * FROM employees LIMIT 20, 10;  -- MySQL独自（LIMIT offset, count）
```

### SQL Server

```sql
-- SQL ServerではTOP句を使う
SELECT TOP 10 * FROM employees
ORDER BY salary DESC;

-- OFFSET-FETCH構文（SQL Server 2012以降）
SELECT * FROM employees
ORDER BY salary DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Oracle

```sql
-- Oracle 12c以降
SELECT * FROM employees
ORDER BY salary DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle 11g以前
SELECT * FROM (
    SELECT e.*, ROWNUM rn FROM employees e
    WHERE ROWNUM <= 30
) WHERE rn > 20;
```

## ORDER BYとLIMITの組み合わせパターン

### トップN件の取得

```sql
-- 給与トップ5
SELECT name, salary FROM employees
ORDER BY salary DESC
LIMIT 5;
```

### 最新N件の取得

```sql
-- 直近に入社した3名
SELECT name, hire_date FROM employees
ORDER BY hire_date DESC
LIMIT 3;
```

### ランダムな1件を取得

```sql
-- PostgreSQL
SELECT * FROM employees ORDER BY RANDOM() LIMIT 1;

-- MySQL
SELECT * FROM employees ORDER BY RAND() LIMIT 1;

-- SQL Server
SELECT TOP 1 * FROM employees ORDER BY NEWID();
```

## SQLの処理順序

SELECT文の各句は以下の順序で処理されます：

```
1. FROM        - テーブルを指定
2. WHERE       - 行を絞り込む
3. GROUP BY    - グループ化する（第10回で解説）
4. HAVING      - グループを絞り込む（第10回で解説）
5. SELECT      - 表示するカラムを選択
6. DISTINCT    - 重複を排除
7. ORDER BY    - 並び替え
8. LIMIT       - 取得件数を制限
```

> 💡 **ポイント**：ORDER BYはSELECTの後に処理されるため、SELECT句で定義した別名が使えます。一方、WHERE句はSELECTの前に処理されるため、別名は使えません。

```sql
-- ✅ ORDER BYで別名を使える
SELECT salary * 12 AS annual_salary FROM employees
ORDER BY annual_salary DESC;

-- ❌ WHEREでは別名を使えない
SELECT salary * 12 AS annual_salary FROM employees
WHERE annual_salary > 5000000;  -- エラー
```

## 実践練習

```sql
-- 練習1：入社日が新しい順に全社員を表示
SELECT name, hire_date FROM employees
ORDER BY hire_date DESC;

-- 練習2：部署ごとに名前の五十音順で表示
SELECT department, name FROM employees
ORDER BY department, name;

-- 練習3：給与の高い方から3～5番目の社員を表示
SELECT name, salary FROM employees
ORDER BY salary DESC
LIMIT 3 OFFSET 2;

-- 練習4：年齢が若い順に上位3名の名前・年齢・部署を表示
SELECT name, age, department FROM employees
ORDER BY age ASC
LIMIT 3;
```

## まとめ

| 機能 | 構文 | 例 |
|------|------|-----|
| 昇順ソート | `ORDER BY col ASC` | `ORDER BY age ASC` |
| 降順ソート | `ORDER BY col DESC` | `ORDER BY salary DESC` |
| 複数キーソート | `ORDER BY col1, col2` | `ORDER BY dept, name` |
| 件数制限 | `LIMIT N` | `LIMIT 10` |
| スキップ | `OFFSET N` | `LIMIT 10 OFFSET 20` |
| NULL位置指定 | `NULLS FIRST/LAST` | `ORDER BY col NULLS LAST` |

次回は、COUNT・SUM・AVG・MAX・MINなどの**集約関数**を学びます。
