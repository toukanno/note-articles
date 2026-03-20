# 【SQL入門】第7回：JOIN入門 — テーブルを結合しよう（INNER JOIN）

## はじめに

実際のデータベースでは、データは複数のテーブルに分けて管理されています。今回は**JOIN（結合）**の基本である**INNER JOIN**を使って、複数のテーブルからデータを取得する方法を学びます。

## なぜテーブルを分けるのか？

### 悪い例：1つのテーブルにすべてを詰め込む

```
┌──────┬──────────┬────────┬──────────┬───────────┐
│emp_id│ name     │ dept   │ dept_loc │ dept_head │
├──────┼──────────┼────────┼──────────┼───────────┤
│ 1    │ 田中太郎 │ 営業部 │ 東京     │ 鈴木部長  │
│ 2    │ 鈴木花子 │ 開発部 │ 大阪     │ 山田部長  │
│ 3    │ 佐藤次郎 │ 営業部 │ 東京     │ 鈴木部長  │ ← 重複！
│ 4    │ 山田美咲 │ 開発部 │ 大阪     │ 山田部長  │ ← 重複！
└──────┴──────────┴────────┴──────────┴───────────┘
```

問題点：
- **データの重複**：部署情報が社員ごとに重複する
- **更新の手間**：部署の場所が変わったら全行を更新する必要がある
- **不整合のリスク**：一部だけ更新すると不整合が発生する

### 良い例：テーブルを分ける（正規化）

```
employees テーブル              departments テーブル
┌──────┬──────────┬───────┐   ┌───────┬────────┬──────┬──────────┐
│emp_id│ name     │dept_id│   │dept_id│ name   │ loc  │ head     │
├──────┼──────────┼───────┤   ├───────┼────────┼──────┼──────────┤
│ 1    │ 田中太郎 │ 1     │   │ 1     │ 営業部 │ 東京 │ 鈴木部長 │
│ 2    │ 鈴木花子 │ 2     │   │ 2     │ 開発部 │ 大阪 │ 山田部長 │
│ 3    │ 佐藤次郎 │ 1     │   │ 3     │ 人事部 │ 東京 │ 高橋部長 │
│ 4    │ 山田美咲 │ 2     │   │ 4     │ 経理部 │ 福岡 │ 佐藤部長 │
└──────┴──────────┴───────┘   └───────┴────────┴──────┴──────────┘
```

→ `dept_id`を使ってテーブル同士を関連付ける

## 練習用テーブルの準備

```sql
-- 部署テーブル
CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL,
    location VARCHAR(50),
    head_name VARCHAR(100)
);

INSERT INTO departments VALUES (1, '営業部', '東京', '鈴木部長');
INSERT INTO departments VALUES (2, '開発部', '大阪', '山田部長');
INSERT INTO departments VALUES (3, '人事部', '東京', '高橋部長');
INSERT INTO departments VALUES (4, '経理部', '福岡', '佐藤部長');
INSERT INTO departments VALUES (5, '広報部', '名古屋', '木村部長');

-- 社員テーブル（dept_idを追加）
CREATE TABLE emp (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    dept_id INTEGER,
    salary INTEGER,
    hire_date DATE
);

INSERT INTO emp VALUES (1, '田中太郎', 1, 450000, '2020-04-01');
INSERT INTO emp VALUES (2, '鈴木花子', 2, 500000, '2022-04-01');
INSERT INTO emp VALUES (3, '佐藤次郎', 3, 480000, '2018-04-01');
INSERT INTO emp VALUES (4, '山田美咲', 2, 520000, '2021-04-01');
INSERT INTO emp VALUES (5, '高橋一郎', 4, 470000, '2019-04-01');
INSERT INTO emp VALUES (6, '伊藤由美', 1, 440000, '2023-04-01');
INSERT INTO emp VALUES (7, '渡辺健太', 2, 600000, '2015-04-01');
INSERT INTO emp VALUES (8, '中村さくら', NULL, 380000, '2024-04-01');
```

> 📝 中村さくらの`dept_id`はNULL（未配属）です。

## INNER JOIN の基本

INNER JOINは、**両方のテーブルで条件に一致する行だけ**を返します。

### 基本構文

```sql
SELECT カラム名
FROM テーブル1
INNER JOIN テーブル2
    ON テーブル1.カラム = テーブル2.カラム;
```

### 基本的な使い方

```sql
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署名,
    d.location AS 勤務地
FROM emp e
INNER JOIN departments d
    ON e.dept_id = d.dept_id;
```

結果：

```
社員名     | 部署名 | 勤務地
-----------+--------+-------
田中太郎   | 営業部 | 東京
鈴木花子   | 開発部 | 大阪
佐藤次郎   | 人事部 | 東京
山田美咲   | 開発部 | 大阪
高橋一郎   | 経理部 | 福岡
伊藤由美   | 営業部 | 東京
渡辺健太   | 開発部 | 大阪
```

> 📝 中村さくら（dept_id=NULL）と広報部（dept_id=5）は結果に含まれていません。INNER JOINは一致するデータのみを返します。

## テーブルの別名（エイリアス）

JOINを使うときは、テーブルに短い別名をつけると便利です。

```sql
-- 別名なし（冗長）
SELECT employees.name, departments.dept_name
FROM employees
INNER JOIN departments
    ON employees.dept_id = departments.dept_id;

-- 別名あり（簡潔）
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d
    ON e.dept_id = d.dept_id;
```

## JOIN + WHERE + ORDER BY

JOINの結果に対して、さらにWHEREやORDER BYを適用できます。

```sql
-- 東京勤務の社員を給与の高い順に表示
SELECT
    e.name,
    d.dept_name,
    d.location,
    e.salary
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = '東京'
ORDER BY e.salary DESC;
```

結果：

```
name     | dept_name | location | salary
---------+-----------+----------+-------
佐藤次郎 | 人事部    | 東京     | 480000
田中太郎 | 営業部    | 東京     | 450000
伊藤由美 | 営業部    | 東京     | 440000
```

## 3つ以上のテーブルを結合

```sql
-- プロジェクトテーブルを追加
CREATE TABLE projects (
    project_id INTEGER PRIMARY KEY,
    project_name VARCHAR(100),
    dept_id INTEGER
);

CREATE TABLE project_members (
    emp_id INTEGER,
    project_id INTEGER,
    role VARCHAR(50)
);
```

```sql
-- 3テーブルの結合
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署,
    p.project_name AS プロジェクト,
    pm.role AS 役割
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
INNER JOIN project_members pm ON e.emp_id = pm.emp_id
INNER JOIN projects p ON pm.project_id = p.project_id
ORDER BY p.project_name, e.name;
```

## 自己結合（Self Join）

同じテーブル同士を結合することもできます。

```sql
-- マネージャーカラムがある場合
CREATE TABLE staff (
    staff_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER
);

INSERT INTO staff VALUES (1, '社長', NULL);
INSERT INTO staff VALUES (2, '部長A', 1);
INSERT INTO staff VALUES (3, '部長B', 1);
INSERT INTO staff VALUES (4, '課長A', 2);
INSERT INTO staff VALUES (5, '社員X', 4);

-- 社員とその上司を表示
SELECT
    s.name AS 社員,
    m.name AS 上司
FROM staff s
INNER JOIN staff m ON s.manager_id = m.staff_id;
```

結果：

```
社員   | 上司
-------+------
部長A  | 社長
部長B  | 社長
課長A  | 部長A
社員X  | 課長A
```

> 📝 社長は`manager_id`がNULLなので、INNER JOINでは結果に含まれません。

## 結合条件の種類

### 等結合（Equi Join）

最も一般的。`=`で結合します。

```sql
ON e.dept_id = d.dept_id
```

### 非等結合（Non-Equi Join）

`=`以外の演算子で結合します。

```sql
-- 給与範囲テーブルとの結合
CREATE TABLE salary_grades (
    grade VARCHAR(10),
    min_salary INTEGER,
    max_salary INTEGER
);

SELECT e.name, e.salary, g.grade
FROM emp e
INNER JOIN salary_grades g
    ON e.salary BETWEEN g.min_salary AND g.max_salary;
```

## 暗黙的JOIN（旧構文）

WHERE句で結合条件を書く古い構文もあります。

```sql
-- 旧構文（非推奨）
SELECT e.name, d.dept_name
FROM emp e, departments d
WHERE e.dept_id = d.dept_id;

-- 新構文（推奨）
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

> ⚠️ 旧構文は可読性が低く、WHERE句を忘れるとクロス結合（全組み合わせ）になってしまうため、**INNER JOIN構文の使用を推奨**します。

## クロス結合（CROSS JOIN）

すべての行の組み合わせを返します。結合条件はありません。

```sql
SELECT e.name, d.dept_name
FROM emp e
CROSS JOIN departments d;
-- emp 8行 × departments 5行 = 40行
```

使用例：カレンダーの生成など

```sql
-- 年と月の組み合わせを生成
SELECT y.year, m.month
FROM (SELECT 2024 AS year UNION SELECT 2025) y
CROSS JOIN (SELECT 1 AS month UNION SELECT 2 UNION SELECT 3) m;
```

## JOINのパフォーマンス

- **結合キーにインデックスを作成**する → 検索が高速化
- 不要なカラムはSELECTしない
- 結合前にWHEREで絞り込む

```sql
-- ✅ 効率的：先に絞り込んでから結合
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary >= 500000;
```

## 実践練習

```sql
-- 練習1：社員名と部署名を表示
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- 練習2：大阪勤務の社員名と給与を表示
SELECT e.name, e.salary, d.location
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = '大阪';

-- 練習3：部署ごとの平均給与を表示
SELECT d.dept_name, AVG(e.salary) AS avg_salary
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;

-- 練習4：社員数が2名以上の部署を表示
SELECT d.dept_name, COUNT(*) AS member_count
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name
HAVING COUNT(*) >= 2;
```

## まとめ

| 概念 | 説明 |
|------|------|
| INNER JOIN | 両テーブルで一致する行のみ返す |
| ON句 | 結合条件を指定 |
| テーブル別名 | `FROM emp e` のように短い名前をつける |
| 自己結合 | 同じテーブルを2回参照して結合 |
| CROSS JOIN | すべての行の組み合わせを返す |

次回は、INNER JOINでは取得できなかった「一致しないデータ」も取得できる**外部結合（LEFT JOIN / RIGHT JOIN / FULL JOIN）**を学びます。
