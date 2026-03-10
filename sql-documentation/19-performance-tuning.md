# 【SQL入門】第19回：パフォーマンスチューニング — SQLを速くする技術

## はじめに

データ量が増えるにつれ、SQLの実行速度は重要な課題になります。今回は、SQLクエリを高速化するための実践的なテクニックを解説します。

## パフォーマンス問題の見つけ方

### EXPLAIN ANALYZE

最も重要なツールです。クエリの実行計画と実際の実行時間を確認できます。

```sql
-- PostgreSQL
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM emp e
JOIN departments d ON e.dept_id = d.dept_id
WHERE e.salary > 500000;
```

出力例：

```
Hash Join  (cost=1.09..2.20 rows=1 width=64) (actual time=0.03..0.04 rows=3 loops=1)
  Hash Cond: (e.dept_id = d.dept_id)
  ->  Seq Scan on emp e  (cost=0.00..1.10 rows=3 width=40) (actual time=0.01..0.01 rows=3 loops=1)
        Filter: (salary > 500000)
        Rows Removed by Filter: 5
  ->  Hash  (cost=1.05..1.05 rows=5 width=36) (actual time=0.01..0.01 rows=5 loops=1)
        ->  Seq Scan on departments d  (cost=0.00..1.05 rows=5 width=36)
Planning Time: 0.10 ms
Execution Time: 0.06 ms
```

### 確認すべきポイント

| 項目 | 良い | 悪い |
|------|------|------|
| Scan Type | Index Scan | Seq Scan（大テーブル） |
| Rows | 実際の行数と推定値が近い | 大きく乖離 |
| Execution Time | 短い | 長い |

### MySQL

```sql
-- 基本的な実行計画
EXPLAIN SELECT * FROM employees WHERE salary > 500000;

-- 詳細な実行計画
EXPLAIN ANALYZE SELECT * FROM employees WHERE salary > 500000;
```

## スロークエリの特定

### PostgreSQL

```sql
-- pg_stat_statementsで遅いクエリを特定
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_time_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### MySQL

```sql
-- スロークエリログの有効化
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 1秒以上のクエリを記録
```

## インデックスによる最適化

### 適切なインデックスの追加

```sql
-- WHERE句でよく使うカラム
CREATE INDEX idx_emp_dept ON employees (department);
CREATE INDEX idx_emp_salary ON employees (salary);

-- JOINで使うカラム（外部キー）
CREATE INDEX idx_emp_dept_id ON emp (dept_id);

-- 複合インデックス（複数条件の検索）
CREATE INDEX idx_emp_dept_salary ON employees (department, salary);
```

### インデックスの効果を確認

```sql
-- Before
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = '開発部';
-- → Seq Scan ... actual time=0.5ms

-- After（インデックス追加後）
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = '開発部';
-- → Index Scan ... actual time=0.05ms
```

### 不要なインデックスの削除

使われていないインデックスは、INSERT/UPDATE/DELETEの速度を低下させます。

```sql
-- PostgreSQL：インデックスの使用状況
SELECT
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan ASC;
```

## SQLの書き方による最適化

### 1. SELECT * を避ける

```sql
-- ❌ 全カラム取得
SELECT * FROM employees WHERE department = '開発部';

-- ✅ 必要なカラムだけ
SELECT name, salary FROM employees WHERE department = '開発部';
```

### 2. 適切な結合方法を選ぶ

```sql
-- 存在チェックにはEXISTSを使う（INより効率的な場合が多い）
-- ❌
SELECT * FROM departments
WHERE dept_id IN (SELECT dept_id FROM emp);

-- ✅
SELECT * FROM departments d
WHERE EXISTS (SELECT 1 FROM emp e WHERE e.dept_id = d.dept_id);
```

### 3. OR を UNION ALL に置き換える

```sql
-- ❌ ORはインデックスが効きにくい場合がある
SELECT * FROM employees
WHERE department = '営業部' OR salary > 500000;

-- ✅ UNION ALLでそれぞれにインデックスを効かせる
SELECT * FROM employees WHERE department = '営業部'
UNION ALL
SELECT * FROM employees WHERE salary > 500000 AND department != '営業部';
```

### 4. サブクエリをJOINに書き換える

```sql
-- ❌ 相関サブクエリ（行ごとに実行される）
SELECT name, salary,
    (SELECT AVG(salary) FROM employees e2
     WHERE e2.department = e1.department) AS dept_avg
FROM employees e1;

-- ✅ JOINに書き換え（1回で計算）
SELECT e.name, e.salary, d.dept_avg
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS dept_avg
    FROM employees
    GROUP BY department
) d ON e.department = d.department;
```

### 5. WHERE句で関数を避ける

```sql
-- ❌ インデックスが効かない
SELECT * FROM employees WHERE YEAR(hire_date) = 2020;
SELECT * FROM employees WHERE UPPER(name) = 'TANAKA';

-- ✅ インデックスが効く
SELECT * FROM employees
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';

-- または式インデックスを作成
CREATE INDEX idx_name_upper ON employees (UPPER(name));
```

### 6. LIMIT を活用する

```sql
-- 存在チェック
-- ❌ 全件カウント
SELECT COUNT(*) FROM employees WHERE department = '開発部';

-- ✅ 1件あるかだけ確認
SELECT 1 FROM employees WHERE department = '開発部' LIMIT 1;
```

### 7. バッチ処理

```sql
-- ❌ 1行ずつINSERT
INSERT INTO logs VALUES (1, 'log1');
INSERT INTO logs VALUES (2, 'log2');
INSERT INTO logs VALUES (3, 'log3');

-- ✅ まとめてINSERT
INSERT INTO logs VALUES
    (1, 'log1'),
    (2, 'log2'),
    (3, 'log3');
```

## テーブル設計による最適化

### 正規化 vs 非正規化

```sql
-- 正規化（データの整合性重視）
-- employeesテーブルとdepartmentsテーブルを分ける
-- → JOINが必要だが、データの更新が容易

-- 非正規化（パフォーマンス重視）
-- 頻繁にJOINされるカラムを1テーブルにまとめる
-- → JOINが不要で高速だが、データの冗長性が増す
```

### パーティショニング

大規模テーブルを分割して、検索対象を限定します。

```sql
-- PostgreSQLのパーティショニング（範囲）
CREATE TABLE access_logs (
    log_id BIGSERIAL,
    log_date DATE NOT NULL,
    path VARCHAR(500),
    status_code INTEGER
) PARTITION BY RANGE (log_date);

-- 月ごとのパーティション
CREATE TABLE access_logs_2024_01 PARTITION OF access_logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE access_logs_2024_02 PARTITION OF access_logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

## 接続とキャッシュの最適化

### コネクションプーリング

データベースへの接続は重い処理です。接続を使い回すことで高速化します。

```
アプリケーション → コネクションプール → データベース
                  (接続を再利用)
```

### クエリキャッシュ

同じクエリの結果をキャッシュして再利用します（アプリケーション側で実装するのが一般的）。

## チューニングのチェックリスト

1. **EXPLAIN ANALYZEで実行計画を確認**したか？
2. **Seq Scanが大テーブルで発生**していないか？
3. **適切なインデックス**が存在するか？
4. **SELECT \*** を使っていないか？
5. **N+1問題**（ループ内でSQLを発行）が発生していないか？
6. **不要なORDER BY**がないか？
7. **大量データのOFFSET**を使っていないか？
8. **WHERE句でカラムに関数**を適用していないか？
9. **統計情報**は最新か？

```sql
-- PostgreSQL：統計情報の更新
ANALYZE employees;

-- MySQL：統計情報の更新
ANALYZE TABLE employees;
```

## 実践練習

```sql
-- 練習1：EXPLAINで実行計画を確認
EXPLAIN ANALYZE
SELECT * FROM employees WHERE department = '開発部' AND salary > 500000;

-- 練習2：インデックスを追加して改善
CREATE INDEX idx_emp_dept_sal ON employees (department, salary);

EXPLAIN ANALYZE
SELECT * FROM employees WHERE department = '開発部' AND salary > 500000;
-- Before と After の実行計画を比較

-- 練習3：非効率なクエリを改善
-- Before
SELECT * FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);

-- After
SELECT e.*
FROM employees e
JOIN (SELECT department, AVG(salary) AS avg_sal FROM employees GROUP BY department) d
ON e.department = d.department AND e.salary > d.avg_sal;
```

## まとめ

| カテゴリ | テクニック |
|---------|----------|
| 分析 | EXPLAIN ANALYZE、スロークエリログ |
| インデックス | 適切なインデックスの追加・削除 |
| SQL書き方 | SELECT *回避、EXISTS活用、関数回避 |
| 設計 | 適切な正規化、パーティショニング |
| インフラ | コネクションプーリング、キャッシュ |

次回（最終回）は、**セキュリティとベストプラクティス**を学びます。
