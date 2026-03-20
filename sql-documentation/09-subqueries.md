# 【SQL入門】第9回：サブクエリ — SQLの中にSQLを書く

## はじめに

今回は**サブクエリ（副問い合わせ）**を学びます。サブクエリとは、SQL文の中に埋め込まれた別のSQL文のことです。複雑な条件を簡潔に表現できる強力な機能です。

## サブクエリの基本

```sql
SELECT カラム名
FROM テーブル名
WHERE カラム名 演算子 (SELECT ...);
```

括弧内のSELECT文が**サブクエリ（内側のクエリ）**、外側のSELECT文が**メインクエリ（外側のクエリ）**です。

## WHERE句でのサブクエリ

### 単一値を返すサブクエリ（スカラサブクエリ）

```sql
-- 平均給与より高い給与の社員を取得
SELECT name, salary
FROM emp
WHERE salary > (SELECT AVG(salary) FROM emp);
```

実行の流れ：
1. サブクエリ `SELECT AVG(salary) FROM emp` → 480000
2. メインクエリ `WHERE salary > 480000` で絞り込み

結果：

```
name     | salary
---------+-------
鈴木花子 | 500000
山田美咲 | 520000
渡辺健太 | 600000
```

### 複数値を返すサブクエリ

#### INとサブクエリ

```sql
-- 東京にある部署の社員を取得
SELECT name, dept_id
FROM emp
WHERE dept_id IN (
    SELECT dept_id FROM departments
    WHERE location = '東京'
);
```

#### NOT INとサブクエリ

```sql
-- プロジェクトに参加していない社員
SELECT name
FROM emp
WHERE emp_id NOT IN (
    SELECT emp_id FROM project_members
);
```

> ⚠️ サブクエリの結果にNULLが含まれると`NOT IN`は空の結果を返します。`NOT EXISTS`を使うのが安全です。

### 比較演算子 + ANY / ALL

#### ANY（SOME）

サブクエリの結果のいずれかと条件を満たす場合にTRUE。

```sql
-- 営業部のいずれかの社員より給与が高い社員
SELECT name, salary
FROM emp
WHERE salary > ANY (
    SELECT salary FROM emp WHERE dept_id = 1
);
-- 営業部の最低給与（440000）より高い社員が対象
```

#### ALL

サブクエリの結果のすべてと条件を満たす場合にTRUE。

```sql
-- 営業部のすべての社員より給与が高い社員
SELECT name, salary
FROM emp
WHERE salary > ALL (
    SELECT salary FROM emp WHERE dept_id = 1
);
-- 営業部の最高給与（450000）より高い社員が対象
```

## EXISTS：存在チェック

サブクエリが**1行以上の結果を返すか**を確認します。

```sql
-- 社員が存在する部署のみ取得
SELECT dept_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM emp e
    WHERE e.dept_id = d.dept_id
);
```

結果：

```
dept_name
---------
営業部
開発部
人事部
経理部
```

### NOT EXISTS

```sql
-- 社員が一人もいない部署
SELECT dept_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM emp e
    WHERE e.dept_id = d.dept_id
);
```

結果：

```
dept_name
---------
広報部
```

### EXISTSとINの違い

| 比較 | EXISTS | IN |
|------|--------|-----|
| NULLの扱い | 問題なし | NOT INでNULLに注意 |
| パフォーマンス | 外側が小さい時に有利 | 内側が小さい時に有利 |
| 可読性 | やや複雑 | シンプル |

## FROM句でのサブクエリ（インラインビュー / 導出テーブル）

FROM句にサブクエリを書くと、一時的なテーブルとして使えます。

```sql
-- 部署ごとの平均給与を計算し、それが全体平均を超える部署
SELECT dept_avg.dept_id, dept_avg.avg_salary
FROM (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM emp
    GROUP BY dept_id
) AS dept_avg
WHERE dept_avg.avg_salary > (SELECT AVG(salary) FROM emp);
```

> 📝 FROM句のサブクエリには必ず**別名（AS）**をつける必要があります。

## SELECT句でのサブクエリ（スカラサブクエリ）

SELECT句にサブクエリを書くと、各行に対して計算結果を付加できます。

```sql
SELECT
    name,
    salary,
    (SELECT AVG(salary) FROM emp) AS 全体平均,
    salary - (SELECT AVG(salary) FROM emp) AS 平均との差
FROM emp;
```

結果：

```
name       | salary | 全体平均 | 平均との差
-----------+--------+---------+-----------
田中太郎   | 450000 | 480000  | -30000
鈴木花子   | 500000 | 480000  |  20000
渡辺健太   | 600000 | 480000  | 120000
...
```

## 相関サブクエリ

メインクエリの値を参照するサブクエリです。行ごとにサブクエリが実行されます。

```sql
-- 自分の部署の平均給与より高い給与の社員
SELECT e1.name, e1.salary, e1.dept_id
FROM emp e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM emp e2
    WHERE e2.dept_id = e1.dept_id  -- メインクエリのe1を参照
);
```

### 相関サブクエリの実行イメージ

```
emp_id=1（田中太郎、dept_id=1）の処理：
  → サブクエリ: SELECT AVG(salary) FROM emp WHERE dept_id = 1 → 445000
  → 450000 > 445000 → TRUE（結果に含む）

emp_id=2（鈴木花子、dept_id=2）の処理：
  → サブクエリ: SELECT AVG(salary) FROM emp WHERE dept_id = 2 → 540000
  → 500000 > 540000 → FALSE（結果に含まない）
...
```

## CTE（Common Table Expression / WITH句）

サブクエリに名前をつけて、見通しの良いクエリを書く方法です。

```sql
-- WITH句を使った書き方
WITH dept_stats AS (
    SELECT
        dept_id,
        AVG(salary) AS avg_salary,
        COUNT(*) AS member_count
    FROM emp
    GROUP BY dept_id
)
SELECT
    d.dept_name,
    ds.avg_salary,
    ds.member_count
FROM dept_stats ds
INNER JOIN departments d ON ds.dept_id = d.dept_id
WHERE ds.avg_salary > 450000;
```

### 複数のCTE

```sql
WITH
high_salary AS (
    SELECT * FROM emp WHERE salary >= 500000
),
tokyo_dept AS (
    SELECT * FROM departments WHERE location = '東京'
)
SELECT h.name, t.dept_name
FROM high_salary h
INNER JOIN tokyo_dept t ON h.dept_id = t.dept_id;
```

### CTEのメリット

- クエリが読みやすくなる
- 同じサブクエリを複数回参照できる
- 再帰クエリが書ける（後述）

## 再帰CTE

階層構造データ（組織図、カテゴリツリーなど）を処理できます。

```sql
-- 組織の階層を再帰的に取得
WITH RECURSIVE org_tree AS (
    -- 基底ケース：トップ（社長）
    SELECT staff_id, name, manager_id, 1 AS level
    FROM staff
    WHERE manager_id IS NULL

    UNION ALL

    -- 再帰ケース：部下を辿る
    SELECT s.staff_id, s.name, s.manager_id, t.level + 1
    FROM staff s
    INNER JOIN org_tree t ON s.manager_id = t.staff_id
)
SELECT
    REPEAT('  ', level - 1) || name AS org_chart,
    level
FROM org_tree
ORDER BY level, name;
```

結果：

```
org_chart    | level
-------------+------
社長         | 1
  部長A      | 2
  部長B      | 2
    課長A    | 3
      社員X  | 4
```

## サブクエリの使い分けガイド

| 用途 | 推奨する方法 |
|------|-------------|
| 単一値との比較 | WHERE句スカラサブクエリ |
| リスト内の確認 | IN + サブクエリ |
| 存在確認 | EXISTS |
| 一時的なテーブル | FROM句サブクエリ or CTE |
| 各行への値付加 | SELECT句スカラサブクエリ |
| 複雑なクエリの整理 | CTE（WITH句） |
| 階層構造の処理 | 再帰CTE |

## 実践練習

```sql
-- 練習1：最高給与の社員を取得
SELECT name, salary FROM emp
WHERE salary = (SELECT MAX(salary) FROM emp);

-- 練習2：自分の部署の平均より高給与の社員
SELECT e.name, e.salary, e.dept_id
FROM emp e
WHERE salary > (
    SELECT AVG(salary) FROM emp WHERE dept_id = e.dept_id
);

-- 練習3：CTEを使って部署統計を見やすく表示
WITH dept_summary AS (
    SELECT dept_id, COUNT(*) AS cnt, AVG(salary) AS avg_sal
    FROM emp GROUP BY dept_id
)
SELECT d.dept_name, ds.cnt, ROUND(ds.avg_sal) AS avg_salary
FROM dept_summary ds
JOIN departments d ON ds.dept_id = d.dept_id;

-- 練習4：社員のいない部署をEXISTSで取得
SELECT d.dept_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM emp e WHERE e.dept_id = d.dept_id
);
```

## まとめ

| サブクエリの種類 | 配置場所 | 返す値 |
|-----------------|---------|--------|
| スカラサブクエリ | WHERE / SELECT | 単一値 |
| リストサブクエリ | WHERE (IN) | 複数値 |
| EXISTS | WHERE | TRUE/FALSE |
| インラインビュー | FROM | テーブル |
| CTE | WITH句 | 名前付きテーブル |
| 相関サブクエリ | WHERE / SELECT | 行ごとに計算 |

次回は、**GROUP BYとHAVING**でデータをグループ化して集計する方法を学びます。
