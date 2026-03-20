# 【SQL入門】第10回：GROUP BYとHAVING — データをグループ化する

## はじめに

第5回で学んだ集約関数は、テーブル全体に対して計算を行いました。今回は**GROUP BY**を使って、データをグループ単位で集計する方法と、**HAVING**でグループを絞り込む方法を解説します。

## GROUP BYの基本

```sql
SELECT カラム名, 集約関数
FROM テーブル名
GROUP BY カラム名;
```

### 部署ごとの社員数

```sql
SELECT department, COUNT(*) AS 人数
FROM employees
GROUP BY department;
```

結果：

```
department | 人数
-----------+-----
営業部     | 2
開発部     | 3
人事部     | 2
経理部     | 1
```

### 処理のイメージ

```
元データ:
田中太郎  営業部  450000
鈴木花子  開発部  500000
佐藤次郎  人事部  480000
山田美咲  開発部  520000
高橋一郎  経理部  470000
伊藤由美  営業部  440000
渡辺健太  開発部  600000
中村さくら 人事部  380000

↓ GROUP BY department

グループ「営業部」: 田中太郎(450000), 伊藤由美(440000)
グループ「開発部」: 鈴木花子(500000), 山田美咲(520000), 渡辺健太(600000)
グループ「人事部」: 佐藤次郎(480000), 中村さくら(380000)
グループ「経理部」: 高橋一郎(470000)

↓ 各グループに集約関数を適用

営業部  COUNT=2  SUM=890000   AVG=445000
開発部  COUNT=3  SUM=1620000  AVG=540000
人事部  COUNT=2  SUM=860000   AVG=430000
経理部  COUNT=1  SUM=470000   AVG=470000
```

## 集約関数との組み合わせ

```sql
-- 部署ごとの統計情報
SELECT
    department,
    COUNT(*) AS 人数,
    SUM(salary) AS 給与合計,
    AVG(salary) AS 平均給与,
    MAX(salary) AS 最高給与,
    MIN(salary) AS 最低給与
FROM employees
GROUP BY department;
```

結果：

```
department | 人数 | 給与合計 | 平均給与 | 最高給与 | 最低給与
-----------+------+---------+---------+---------+---------
営業部     | 2    |  890000 | 445000  |  450000 |  440000
開発部     | 3    | 1620000 | 540000  |  600000 |  500000
人事部     | 2    |  860000 | 430000  |  480000 |  380000
経理部     | 1    |  470000 | 470000  |  470000 |  470000
```

## 複数カラムでのGROUP BY

```sql
-- 部署と入社年ごとの集計
SELECT
    department,
    EXTRACT(YEAR FROM hire_date) AS hire_year,
    COUNT(*) AS 人数,
    AVG(salary) AS 平均給与
FROM employees
GROUP BY department, EXTRACT(YEAR FROM hire_date)
ORDER BY department, hire_year;
```

## GROUP BYのルール

### SELECT句に書けるもの

GROUP BYを使う場合、SELECT句に書けるのは：

1. GROUP BYに指定したカラム
2. 集約関数
3. 定数

```sql
-- ✅ 正しい
SELECT department, COUNT(*), AVG(salary)
FROM employees
GROUP BY department;

-- ❌ エラー: nameはGROUP BYにもなく集約関数でもない
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;
```

> 📝 MySQLの`ONLY_FULL_GROUP_BY`モードが無効の場合、上記のエラーは出ませんが、結果が不定になるため推奨しません。

## HAVING句：グループの絞り込み

WHERE句は**行の絞り込み**に使いますが、HAVING句は**グループの絞り込み**に使います。

```sql
SELECT department, COUNT(*) AS 人数
FROM employees
GROUP BY department
HAVING COUNT(*) >= 2;
```

結果：

```
department | 人数
-----------+-----
営業部     | 2
開発部     | 3
人事部     | 2
```

### WHEREとHAVINGの違い

```sql
-- WHERE：グループ化の前に行を絞り込む
-- HAVING：グループ化の後にグループを絞り込む

SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'    -- ①行を絞り込む
GROUP BY department                 -- ②グループ化
HAVING AVG(salary) >= 450000;      -- ③グループを絞り込む
```

処理順序：

```
1. FROM employees            ← テーブルを読む
2. WHERE hire_date >= ...    ← 2020年以降入社の社員に絞る
3. GROUP BY department       ← 部署でグループ化
4. HAVING AVG(salary) >= ... ← 平均給与45万以上のグループに絞る
5. SELECT                    ← 表示するカラムを選択
```

### よくある間違い

```sql
-- ❌ HAVINGに集約関数でない条件を書く（動くがWHEREで書くべき）
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING department = '開発部';

-- ✅ 行の条件はWHEREに書く
SELECT department, AVG(salary)
FROM employees
WHERE department = '開発部'
GROUP BY department;
```

## GROUP BYの応用パターン

### パターン1：日付ごとの集計

```sql
-- 入社年ごとの社員数
SELECT
    EXTRACT(YEAR FROM hire_date) AS 入社年,
    COUNT(*) AS 人数
FROM employees
GROUP BY EXTRACT(YEAR FROM hire_date)
ORDER BY 入社年;
```

### パターン2：範囲ごとの集計

```sql
-- 給与レンジごとの人数
SELECT
    CASE
        WHEN salary < 400000 THEN '40万未満'
        WHEN salary < 500000 THEN '40万〜50万'
        WHEN salary < 600000 THEN '50万〜60万'
        ELSE '60万以上'
    END AS 給与レンジ,
    COUNT(*) AS 人数
FROM employees
GROUP BY
    CASE
        WHEN salary < 400000 THEN '40万未満'
        WHEN salary < 500000 THEN '40万〜50万'
        WHEN salary < 600000 THEN '50万〜60万'
        ELSE '60万以上'
    END
ORDER BY MIN(salary);
```

### パターン3：JOINとGROUP BY

```sql
-- 部署名を表示しながら集計
SELECT
    d.dept_name,
    d.location,
    COUNT(e.emp_id) AS 人数,
    COALESCE(AVG(e.salary), 0) AS 平均給与
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name, d.location
ORDER BY 人数 DESC;
```

### パターン4：条件付き集計

```sql
-- 部署ごとに高給与・低給与の人数を集計
SELECT
    department,
    COUNT(*) AS 全員,
    COUNT(CASE WHEN salary >= 500000 THEN 1 END) AS 高給与,
    COUNT(CASE WHEN salary < 500000 THEN 1 END) AS 低給与
FROM employees
GROUP BY department;
```

## GROUPING SETS / ROLLUP / CUBE

### ROLLUP：小計と合計

```sql
-- 部署ごとの小計と全体の合計を一度に取得
SELECT
    COALESCE(department, '【合計】') AS department,
    COUNT(*) AS 人数,
    SUM(salary) AS 給与合計
FROM employees
GROUP BY ROLLUP(department);
```

結果：

```
department | 人数 | 給与合計
-----------+------+---------
営業部     | 2    |  890000
開発部     | 3    | 1620000
人事部     | 2    |  860000
経理部     | 1    |  470000
【合計】    | 8    | 3840000
```

### CUBE：すべての組み合わせ

```sql
SELECT
    department,
    EXTRACT(YEAR FROM hire_date) AS year,
    COUNT(*) AS 人数
FROM employees
GROUP BY CUBE(department, EXTRACT(YEAR FROM hire_date));
```

### GROUPING SETS：特定の組み合わせ

```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS year, COUNT(*)
FROM employees
GROUP BY GROUPING SETS (
    (department),                           -- 部署ごと
    (EXTRACT(YEAR FROM hire_date)),         -- 年ごと
    ()                                      -- 全体
);
```

## 実践練習

```sql
-- 練習1：部署ごとの平均年齢
SELECT department, ROUND(AVG(age), 1) AS 平均年齢
FROM employees
GROUP BY department;

-- 練習2：2名以上いる部署で平均給与が高い順
SELECT department, COUNT(*) AS 人数, AVG(salary) AS 平均給与
FROM employees
GROUP BY department
HAVING COUNT(*) >= 2
ORDER BY 平均給与 DESC;

-- 練習3：入社年ごとの統計
SELECT
    EXTRACT(YEAR FROM hire_date) AS 年,
    COUNT(*) AS 人数,
    AVG(salary) AS 平均給与
FROM employees
GROUP BY EXTRACT(YEAR FROM hire_date)
ORDER BY 年;

-- 練習4：給与レンジごとの分布
SELECT
    FLOOR(salary / 100000) * 10 || '万台' AS レンジ,
    COUNT(*) AS 人数
FROM employees
GROUP BY FLOOR(salary / 100000)
ORDER BY FLOOR(salary / 100000);
```

## まとめ

| 機能 | 構文 | 用途 |
|------|------|------|
| グループ化 | `GROUP BY col` | データをグループ単位で集計 |
| グループ絞り込み | `HAVING 条件` | 集計結果で絞り込み |
| 小計・合計 | `ROLLUP` | 階層的な集計 |
| 全組み合わせ | `CUBE` | 多次元集計 |
| 指定組み合わせ | `GROUPING SETS` | 特定パターンの集計 |

次回は、複数のクエリ結果を結合する**UNION**を学びます。
