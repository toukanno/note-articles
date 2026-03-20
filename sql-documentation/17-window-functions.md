# 【SQL入門】第17回：ウィンドウ関数 — 高度な分析クエリ

## はじめに

GROUP BYを使うと行がグループに集約されてしまいますが、**ウィンドウ関数**を使えば、各行を保持したまま集計や順位付けができます。データ分析で非常に強力な機能です。

## ウィンドウ関数とは

```
GROUP BY + 集約関数：  行がグループ化される（行数が減る）
ウィンドウ関数：       各行を保持したまま計算結果を追加する（行数が変わらない）
```

## 基本構文

```sql
関数名() OVER (
    [PARTITION BY カラム名]
    [ORDER BY カラム名]
    [フレーム指定]
)
```

- `PARTITION BY`：グループを定義（省略可：全行が1グループ）
- `ORDER BY`：グループ内の順序を定義
- フレーム指定：計算対象の行の範囲を定義

## ROW_NUMBER：行番号

各行に連番を振ります。

```sql
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank
FROM employees;
```

結果：

```
name       | department | salary | rank
-----------+------------+--------+-----
渡辺健太   | 開発部     | 600000 | 1
山田美咲   | 開発部     | 520000 | 2
鈴木花子   | 開発部     | 500000 | 3
佐藤次郎   | 人事部     | 480000 | 4
高橋一郎   | 経理部     | 470000 | 5
田中太郎   | 営業部     | 450000 | 6
伊藤由美   | 営業部     | 440000 | 7
中村さくら | 人事部     | 380000 | 8
```

### PARTITION BYとの組み合わせ

```sql
-- 部署ごとの給与順位
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS dept_rank
FROM employees;
```

結果：

```
name       | department | salary | dept_rank
-----------+------------+--------+----------
田中太郎   | 営業部     | 450000 | 1
伊藤由美   | 営業部     | 440000 | 2
渡辺健太   | 開発部     | 600000 | 1
山田美咲   | 開発部     | 520000 | 2
鈴木花子   | 開発部     | 500000 | 3
佐藤次郎   | 人事部     | 480000 | 1
中村さくら | 人事部     | 380000 | 2
高橋一郎   | 経理部     | 470000 | 1
```

## RANK / DENSE_RANK：順位

```sql
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

同じ値がある場合の動作の違い：

```
salary = 500000 が2人いる場合：
ROW_NUMBER: 1, 2, 3, 4, 5, ...    （常に連番）
RANK:       1, 1, 3, 4, 5, ...    （同順位の次はスキップ）
DENSE_RANK: 1, 1, 2, 3, 4, ...    （同順位の次はスキップしない）
```

## NTILE：均等分割

データをN個のグループに均等分割します。

```sql
-- 給与を4分位に分割
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

結果：

```
name       | salary | quartile
-----------+--------+---------
渡辺健太   | 600000 | 1  ← 上位25%
山田美咲   | 520000 | 1
鈴木花子   | 500000 | 2
佐藤次郎   | 480000 | 2
高橋一郎   | 470000 | 3
田中太郎   | 450000 | 3
伊藤由美   | 440000 | 4
中村さくら | 380000 | 4  ← 下位25%
```

## LAG / LEAD：前後の行の値を参照

### LAG：前の行の値

```sql
SELECT
    name,
    hire_date,
    LAG(name) OVER (ORDER BY hire_date) AS prev_employee,
    LAG(hire_date) OVER (ORDER BY hire_date) AS prev_date
FROM employees
ORDER BY hire_date;
```

結果：

```
name       | hire_date  | prev_employee | prev_date
-----------+------------+--------------+-----------
渡辺健太   | 2015-04-01 | NULL         | NULL
佐藤次郎   | 2018-04-01 | 渡辺健太     | 2015-04-01
高橋一郎   | 2019-04-01 | 佐藤次郎     | 2018-04-01
田中太郎   | 2020-04-01 | 高橋一郎     | 2019-04-01
...
```

### LEAD：次の行の値

```sql
-- 次の入社者との給与差
SELECT
    name,
    salary,
    LEAD(salary) OVER (ORDER BY hire_date) AS next_salary,
    salary - LEAD(salary) OVER (ORDER BY hire_date) AS diff
FROM employees
ORDER BY hire_date;
```

### LAG/LEADのオプション

```sql
-- LAG(カラム, 行数, デフォルト値)
LAG(salary, 1, 0) OVER (ORDER BY hire_date)   -- 1行前、なければ0
LAG(salary, 2) OVER (ORDER BY hire_date)       -- 2行前
LEAD(salary, 1, 0) OVER (ORDER BY hire_date)   -- 1行後、なければ0
```

## FIRST_VALUE / LAST_VALUE

ウィンドウ内の最初/最後の値を取得します。

```sql
-- 部署内の最高給与者
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS top_earner
FROM employees;
```

## 集約関数のウィンドウ版

通常の集約関数もOVER句を付けるとウィンドウ関数として使えます。

```sql
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER () AS 全体平均,
    AVG(salary) OVER (PARTITION BY department) AS 部署平均,
    salary - AVG(salary) OVER (PARTITION BY department) AS 部署平均との差
FROM employees;
```

結果：

```
name       | department | salary | 全体平均 | 部署平均 | 部署平均との差
-----------+------------+--------+---------+---------+--------------
田中太郎   | 営業部     | 450000 | 480000  | 445000  |  5000
伊藤由美   | 営業部     | 440000 | 480000  | 445000  | -5000
渡辺健太   | 開発部     | 600000 | 480000  | 540000  | 60000
山田美咲   | 開発部     | 520000 | 480000  | 540000  | -20000
鈴木花子   | 開発部     | 500000 | 480000  | 540000  | -40000
...
```

### 累計（ランニングトータル）

```sql
-- 入社日順の累計給与
SELECT
    name,
    hire_date,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees
ORDER BY hire_date;
```

結果：

```
name       | hire_date  | salary | running_total
-----------+------------+--------+--------------
渡辺健太   | 2015-04-01 | 600000 |  600000
佐藤次郎   | 2018-04-01 | 480000 | 1080000
高橋一郎   | 2019-04-01 | 470000 | 1550000
田中太郎   | 2020-04-01 | 450000 | 2000000
...
```

## フレーム指定

計算対象の行の範囲を細かく制御できます。

```sql
関数() OVER (
    ORDER BY カラム
    ROWS BETWEEN 開始 AND 終了
)
```

| 指定 | 意味 |
|------|------|
| `UNBOUNDED PRECEDING` | パーティションの最初 |
| `N PRECEDING` | N行前 |
| `CURRENT ROW` | 現在行 |
| `N FOLLOWING` | N行後 |
| `UNBOUNDED FOLLOWING` | パーティションの最後 |

### 移動平均

```sql
-- 3行の移動平均（前の行、現在行、次の行）
SELECT
    name,
    salary,
    AVG(salary) OVER (
        ORDER BY hire_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg
FROM employees
ORDER BY hire_date;
```

### 累計と全体合計

```sql
SELECT
    name,
    salary,
    -- 累計（最初から現在行まで）
    SUM(salary) OVER (
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative,
    -- 全体合計
    SUM(salary) OVER (
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS total
FROM employees
ORDER BY hire_date;
```

## NAMED WINDOW（PostgreSQL）

同じウィンドウ定義を複数回使う場合、名前をつけて再利用できます。

```sql
SELECT
    name,
    department,
    salary,
    RANK() OVER w AS rank,
    AVG(salary) OVER w AS avg_salary
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

## 実践パターン

### 部署ごとのトップN

```sql
-- 各部署の給与トップ2
SELECT * FROM (
    SELECT
        name,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department ORDER BY salary DESC
        ) AS rn
    FROM employees
) ranked
WHERE rn <= 2;
```

### 前月比の計算

```sql
-- 月次売上の前月比
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))::NUMERIC
        / LAG(revenue) OVER (ORDER BY month) * 100, 1
    ) AS growth_rate
FROM monthly_sales;
```

## 実践練習

```sql
-- 練習1：全社員に給与順位をつける
SELECT name, salary,
    RANK() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- 練習2：部署内での給与順位と部署平均
SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;

-- 練習3：入社日順の累計人数
SELECT name, hire_date,
    COUNT(*) OVER (ORDER BY hire_date) AS cumulative_count
FROM employees
ORDER BY hire_date;
```

## まとめ

| 関数 | 用途 |
|------|------|
| `ROW_NUMBER()` | 連番を振る |
| `RANK()` | 順位（同順位後スキップ） |
| `DENSE_RANK()` | 順位（スキップなし） |
| `NTILE(n)` | N分割 |
| `LAG(col, n)` | n行前の値 |
| `LEAD(col, n)` | n行後の値 |
| `FIRST_VALUE(col)` | 最初の値 |
| `SUM/AVG/COUNT OVER()` | ウィンドウ集約 |

次回は、**ストアドプロシージャと関数**を学びます。
