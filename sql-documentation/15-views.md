# 【SQL入門】第15回：ビュー — 仮想テーブルを活用する

## はじめに

複雑なクエリを毎回書くのは大変です。**ビュー（VIEW）**を使えば、クエリに名前をつけて保存し、テーブルのように扱えます。今回はビューの作成と活用方法を解説します。

## ビューとは

ビューは、SELECT文に名前をつけた**仮想的なテーブル**です。ビュー自体にはデータは保存されず、参照されるたびにSELECT文が実行されます。

```
通常のテーブル：  データを実際に保持している
ビュー：         SELECT文を保存しており、参照時に実行される
```

## ビューの作成

### 基本構文

```sql
CREATE VIEW ビュー名 AS
SELECT文;
```

### 基本的なビュー

```sql
-- 開発部の社員ビュー
CREATE VIEW dev_team AS
SELECT emp_id, name, salary, hire_date
FROM employees
WHERE department = '開発部';
```

### ビューの使用

テーブルと同じようにSELECT文で使えます。

```sql
-- ビューからデータを取得
SELECT * FROM dev_team;
```

結果：

```
emp_id | name     | salary | hire_date
-------+----------+--------+-----------
     2 | 鈴木花子 | 500000 | 2022-04-01
     4 | 山田美咲 | 520000 | 2021-04-01
     7 | 渡辺健太 | 600000 | 2015-04-01
```

```sql
-- ビューに対してWHEREやORDER BYも使える
SELECT name, salary FROM dev_team
WHERE salary >= 520000
ORDER BY salary DESC;
```

### 結合を含むビュー

```sql
-- 社員と部署情報を結合したビュー
CREATE VIEW employee_details AS
SELECT
    e.emp_id,
    e.name AS employee_name,
    e.salary,
    e.hire_date,
    d.dept_name,
    d.location
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

```sql
-- 複雑な結合を意識せずに使える
SELECT employee_name, dept_name, location
FROM employee_details
WHERE location = '東京';
```

### 集約を含むビュー

```sql
-- 部署統計ビュー
CREATE VIEW department_stats AS
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS member_count,
    COALESCE(AVG(e.salary), 0) AS avg_salary,
    COALESCE(MAX(e.salary), 0) AS max_salary,
    COALESCE(MIN(e.salary), 0) AS min_salary
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;
```

```sql
SELECT * FROM department_stats ORDER BY avg_salary DESC;
```

## ビューのメリット

### 1. クエリの簡素化

```sql
-- ビューなし：毎回複雑なクエリを書く
SELECT e.name, d.dept_name, d.location, e.salary,
       CASE WHEN e.salary >= 500000 THEN '高' ELSE '標準' END AS level
FROM emp e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = '東京';

-- ビューあり：簡潔に書ける
CREATE VIEW tokyo_employees AS
SELECT e.name, d.dept_name, d.location, e.salary,
       CASE WHEN e.salary >= 500000 THEN '高' ELSE '標準' END AS level
FROM emp e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = '東京';

SELECT * FROM tokyo_employees;
```

### 2. セキュリティ（データの隠蔽）

ユーザーに見せたくないカラムを隠すことができます。

```sql
-- 給与情報を隠したビュー
CREATE VIEW public_employee_info AS
SELECT emp_id, name, department, hire_date
FROM employees;
-- salaryカラムはビューに含まれない

-- 一般ユーザーにはビューのみアクセスを許可
GRANT SELECT ON public_employee_info TO general_user;
```

### 3. ロジックの一元管理

ビジネスロジックをビューに集約し、変更があっても1箇所を修正するだけで済みます。

```sql
-- 社員の評価ランクのロジック
CREATE VIEW employee_rankings AS
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 550000 THEN 'S'
        WHEN salary >= 500000 THEN 'A'
        WHEN salary >= 450000 THEN 'B'
        WHEN salary >= 400000 THEN 'C'
        ELSE 'D'
    END AS rank
FROM employees;
```

## ビューの変更と削除

### ビューの変更

```sql
-- CREATE OR REPLACE（PostgreSQL, MySQL）
CREATE OR REPLACE VIEW dev_team AS
SELECT emp_id, name, salary, hire_date, age
FROM employees
WHERE department = '開発部';
```

### ビューの削除

```sql
DROP VIEW dev_team;
DROP VIEW IF EXISTS dev_team;
```

## ビューを通じたデータ更新

条件を満たすビューに対して、INSERT/UPDATE/DELETEを実行できます。

### 更新可能なビューの条件

- 単一テーブルから作成されている
- 集約関数（COUNT, SUMなど）を含まない
- GROUP BY / HAVING / DISTINCT を含まない
- UNION を含まない

```sql
-- 更新可能なビュー
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE is_active = TRUE;

-- ビュー経由でデータを更新
UPDATE active_employees SET salary = 500000 WHERE emp_id = 1;

-- ビュー経由でデータを挿入
INSERT INTO active_employees (emp_id, name, salary, is_active)
VALUES (20, '新人', 350000, TRUE);
```

### WITH CHECK OPTION

ビューの条件を満たさないデータの挿入/更新を防ぎます。

```sql
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE is_active = TRUE
WITH CHECK OPTION;

-- ✅ is_active = TRUE なので成功
INSERT INTO active_employees (emp_id, name, is_active)
VALUES (21, '社員A', TRUE);

-- ❌ エラー: is_active = FALSE はビューの条件に合わない
INSERT INTO active_employees (emp_id, name, is_active)
VALUES (22, '社員B', FALSE);
```

## マテリアライズドビュー（PostgreSQL）

通常のビューは参照のたびにクエリが実行されますが、**マテリアライズドビュー**は結果を物理的に保存します。

```sql
-- マテリアライズドビューの作成
CREATE MATERIALIZED VIEW mv_department_stats AS
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS member_count,
    AVG(e.salary) AS avg_salary
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;
```

### 通常ビューとマテリアライズドビューの比較

| 項目 | 通常のビュー | マテリアライズドビュー |
|------|-------------|---------------------|
| データ保存 | なし | あり |
| 参照速度 | クエリ実行時間に依存 | 高速（保存済みデータ） |
| データの鮮度 | 常に最新 | リフレッシュが必要 |
| ストレージ | 不要 | 必要 |
| インデックス | 不可 | 可能 |

### リフレッシュ

```sql
-- データを最新に更新
REFRESH MATERIALIZED VIEW mv_department_stats;

-- 読み取りをブロックせずにリフレッシュ（UNIQUEインデックスが必要）
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_department_stats;
```

### マテリアライズドビューの使いどころ

- 計算コストの高い集計クエリ
- レポートやダッシュボード用のデータ
- あまり更新されないマスタデータの結合結果

## ビューの情報を確認

```sql
-- PostgreSQL：ビューの定義を確認
SELECT definition FROM pg_views WHERE viewname = 'dev_team';

-- MySQL：ビューの一覧
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- MySQL：ビューの定義を確認
SHOW CREATE VIEW dev_team;
```

## 実践練習

```sql
-- 練習1：社員詳細ビューを作成
CREATE VIEW v_employee_detail AS
SELECT
    e.emp_id,
    e.name,
    e.salary,
    e.salary * 12 AS annual_salary,
    d.dept_name,
    d.location
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id;

-- 練習2：ビューを使ったクエリ
SELECT * FROM v_employee_detail
WHERE location = '大阪'
ORDER BY annual_salary DESC;

-- 練習3：部署サマリービュー
CREATE VIEW v_dept_summary AS
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS 人数,
    COALESCE(ROUND(AVG(e.salary)), 0) AS 平均給与
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;

SELECT * FROM v_dept_summary ORDER BY 人数 DESC;
```

## まとめ

| 項目 | 内容 |
|------|------|
| ビューの作成 | `CREATE VIEW v AS SELECT ...` |
| ビューの変更 | `CREATE OR REPLACE VIEW v AS ...` |
| ビューの削除 | `DROP VIEW v` |
| 更新可能ビュー | 単一テーブル、集約なし |
| CHECK OPTION | 条件外データの挿入を防止 |
| マテリアライズドビュー | 結果を物理保存（PostgreSQL） |

次回は、データの整合性を保つ**トランザクション**を学びます。
