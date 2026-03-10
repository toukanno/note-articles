# 【SQL入門】第14回：インデックス — 検索を高速化する仕組み

## はじめに

データ量が増えるにつれ、クエリの実行速度が遅くなることがあります。今回は、データベースの検索を劇的に高速化する**インデックス（索引）**を解説します。

## インデックスとは

インデックスは、本の**索引**と同じ概念です。

```
本を読む場合：
  ❌ 目的のページを探すために1ページずつ読む → 遅い（フルスキャン）
  ✅ 巻末の索引で目的のページを調べる → 速い（インデックススキャン）

データベースの場合：
  ❌ テーブルの全行を順にチェックする → 遅い（フルテーブルスキャン）
  ✅ インデックスで対象行の位置を調べる → 速い（インデックススキャン）
```

## インデックスの仕組み（B-Tree）

最も一般的なインデックスは**B-Tree（バランス木）**構造です。

```
                    [500000]
                   /        \
          [400000]            [550000]
         /       \           /        \
   [380000]  [440000-470000]  [500000-520000]  [600000]
      ↓         ↓      ↓         ↓       ↓        ↓
   中村      伊藤   高橋      鈴木    山田     渡辺
```

salary=500000の検索：
1. ルート[500000]と比較 → 一致！ → 2ステップで到達
2. 100万行あっても約20ステップで到達（log₂ 1,000,000 ≈ 20）

## インデックスの作成

### 基本構文

```sql
CREATE INDEX インデックス名 ON テーブル名 (カラム名);
```

### 単一カラムインデックス

```sql
-- 給与での検索を高速化
CREATE INDEX idx_employees_salary ON employees (salary);

-- 部署での検索を高速化
CREATE INDEX idx_employees_department ON employees (department);
```

### 複合インデックス（マルチカラムインデックス）

```sql
-- 部署と給与の組み合わせでの検索を高速化
CREATE INDEX idx_emp_dept_salary ON employees (department, salary);
```

> 💡 **カラムの順序が重要**です。`(department, salary)`のインデックスは：
> - `WHERE department = '開発部'` → ✅ 使われる
> - `WHERE department = '開発部' AND salary > 500000` → ✅ 使われる
> - `WHERE salary > 500000` → ❌ 使われない（先頭カラムが条件にない）

### ユニークインデックス

```sql
-- 値の一意性を保証するインデックス
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

### 部分インデックス（PostgreSQL）

特定の条件に合う行のみにインデックスを作成します。

```sql
-- アクティブな社員だけにインデックス
CREATE INDEX idx_active_emp ON employees (name)
WHERE is_active = TRUE;
```

### 式インデックス

```sql
-- 大文字小文字を区別しない検索用
CREATE INDEX idx_email_lower ON users (LOWER(email));

-- 日付の年部分での検索用
CREATE INDEX idx_hire_year ON employees (EXTRACT(YEAR FROM hire_date));
```

## インデックスの削除

```sql
DROP INDEX idx_employees_salary;

-- PostgreSQL
DROP INDEX IF EXISTS idx_employees_salary;

-- MySQL
DROP INDEX idx_employees_salary ON employees;
-- または
ALTER TABLE employees DROP INDEX idx_employees_salary;
```

## EXPLAIN：クエリの実行計画を確認

インデックスが使われているか確認するには、**EXPLAIN**を使います。

### PostgreSQL

```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 500000;
```

出力例：

```
Seq Scan on employees  (cost=0.00..1.10 rows=3 width=44)
  Filter: (salary > 500000)
  Rows Removed by Filter: 5
  Planning Time: 0.05 ms
  Execution Time: 0.02 ms
```

インデックス作成後：

```
Index Scan using idx_employees_salary on employees
  Index Cond: (salary > 500000)
  Planning Time: 0.08 ms
  Execution Time: 0.01 ms
```

### MySQL

```sql
EXPLAIN SELECT * FROM employees WHERE salary > 500000;
```

出力の見方：

| フィールド | 説明 |
|-----------|------|
| type | アクセスタイプ（ALL=フルスキャン、ref=インデックス参照） |
| possible_keys | 使用可能なインデックス |
| key | 実際に使用されたインデックス |
| rows | 推定読取行数 |

## インデックスが使われるケース・使われないケース

### 使われるケース

```sql
-- 等価検索
WHERE email = 'tanaka@example.com'

-- 範囲検索
WHERE salary BETWEEN 400000 AND 500000
WHERE hire_date >= '2020-01-01'

-- 前方一致のLIKE
WHERE name LIKE '田中%'

-- ORDER BY
ORDER BY salary DESC

-- JOIN条件
ON e.dept_id = d.dept_id
```

### 使われないケース

```sql
-- ❌ 後方一致・部分一致のLIKE
WHERE name LIKE '%太郎'

-- ❌ カラムに関数を適用
WHERE YEAR(hire_date) = 2020

-- ❌ 暗黙の型変換
WHERE emp_id = '1'  -- emp_idがINTEGER型の場合

-- ❌ NOT IN, !=
WHERE department != '開発部'

-- ❌ ORの一部にインデックスがない
WHERE indexed_col = 1 OR non_indexed_col = 2

-- ❌ テーブルの大部分を取得する場合（オプティマイザがフルスキャンを選択）
WHERE salary > 0  -- ほぼ全行が該当
```

## インデックスの種類

| 種類 | 用途 | RDBMS |
|------|------|-------|
| B-Tree | 一般的な検索・ソート | 全RDBMS |
| Hash | 等価検索のみ（高速） | PostgreSQL, MySQL |
| GiST | 地理データ、全文検索 | PostgreSQL |
| GIN | 配列、JSONB、全文検索 | PostgreSQL |
| BRIN | 大規模テーブルの範囲検索 | PostgreSQL |
| 全文検索 | テキスト検索 | MySQL(FULLTEXT) |

```sql
-- PostgreSQL: GINインデックス（JSONB用）
CREATE INDEX idx_data_gin ON documents USING GIN (data);

-- PostgreSQL: BRINインデックス（大規模テーブル用）
CREATE INDEX idx_logs_created ON access_logs USING BRIN (created_at);

-- MySQL: 全文検索インデックス
CREATE FULLTEXT INDEX idx_content ON articles (title, body);
```

## インデックス設計のベストプラクティス

### 1. インデックスを作るべきカラム

- WHERE句で頻繁に使われるカラム
- JOIN条件のカラム（外部キー）
- ORDER BYで使われるカラム
- UNIQUE制約が必要なカラム

### 2. インデックスを作るべきでないカラム

- データ量が少ないテーブル（数百行以下）
- 値の種類が少ないカラム（性別、フラグなど）
- 頻繁にUPDATE/INSERTされるカラム
- ほとんど使われないカラム

### 3. インデックスのデメリット

- **ディスク容量**を消費する
- **INSERT/UPDATE/DELETE**が遅くなる（インデックスの更新が必要）
- メンテナンスが必要（断片化の解消など）

```sql
-- インデックスのサイズを確認（PostgreSQL）
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'employees';

-- インデックスの再構築（PostgreSQL）
REINDEX INDEX idx_employees_salary;

-- インデックスの最適化（MySQL）
OPTIMIZE TABLE employees;
```

### 4. カバリングインデックス

クエリに必要な全カラムがインデックスに含まれていると、テーブル本体にアクセスせずにインデックスだけで結果を返せます。

```sql
-- department と salary だけを取得するクエリが多い場合
CREATE INDEX idx_dept_salary ON employees (department, salary);

-- このクエリはインデックスだけで完結する（Index Only Scan）
SELECT department, salary FROM employees
WHERE department = '開発部';
```

### INCLUDEカラム（PostgreSQL）

```sql
-- 検索条件には使わないが、結果に含めたいカラムをINCLUDEで追加
CREATE INDEX idx_dept_include ON employees (department)
INCLUDE (name, salary);
```

## 実践練習

```sql
-- 練習1：メールアドレスにユニークインデックスを作成
CREATE UNIQUE INDEX idx_emp_email ON employees (email);

-- 練習2：部署と入社日の複合インデックスを作成
CREATE INDEX idx_emp_dept_hire ON employees (department, hire_date);

-- 練習3：EXPLAINでインデックスの効果を確認
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = '開発部';

-- 練習4：インデックスの一覧を確認（PostgreSQL）
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'employees';
```

## まとめ

| 項目 | 内容 |
|------|------|
| 目的 | 検索の高速化 |
| 基本構造 | B-Tree |
| 作成 | `CREATE INDEX idx ON table (col)` |
| 削除 | `DROP INDEX idx` |
| 確認 | `EXPLAIN ANALYZE` |
| メリット | SELECT高速化 |
| デメリット | INSERT/UPDATE遅延、容量消費 |

次回は、複雑なクエリを簡単に使いまわせる**ビュー（VIEW）**を学びます。
