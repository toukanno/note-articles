# 【SQL入門】第20回：セキュリティとベストプラクティス

## はじめに

シリーズ最終回の今回は、SQLを安全に運用するための**セキュリティ対策**と、実務で役立つ**ベストプラクティス**を解説します。

## SQLインジェクション

SQLインジェクションは、最も危険なセキュリティ脆弱性の一つです。悪意のあるSQL文を注入されることで、データの漏洩や改ざんが発生します。

### 脆弱なコード例

```python
# ❌ 危険：ユーザー入力を直接SQL文に埋め込む
user_input = request.get("username")
query = f"SELECT * FROM users WHERE username = '{user_input}'"
cursor.execute(query)
```

攻撃者が以下の入力をした場合：

```
' OR '1'='1' --
```

実行されるSQL：

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' --'
-- 全ユーザーのデータが返される！
```

さらに危険な入力：

```
'; DROP TABLE users; --
```

```sql
SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
-- テーブルが削除される！
```

### 対策1：パラメータ化クエリ（プリペアドステートメント）

```python
# ✅ 安全：パラメータ化クエリを使用
cursor.execute(
    "SELECT * FROM users WHERE username = %s",
    (user_input,)
)
```

```javascript
// Node.js (PostgreSQL)
const result = await pool.query(
    'SELECT * FROM users WHERE username = $1',
    [userInput]
);
```

```java
// Java (JDBC)
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?"
);
stmt.setString(1, userInput);
ResultSet rs = stmt.executeQuery();
```

```ruby
# Ruby on Rails
User.where(username: user_input)
```

### 対策2：入力値のバリデーション

```python
# 入力値の検証
import re

def validate_username(username):
    if not re.match(r'^[a-zA-Z0-9_]{3,30}$', username):
        raise ValueError("無効なユーザー名です")
    return username
```

### 対策3：最小権限の原則

アプリケーション用のDBユーザーには必要最小限の権限のみ付与します。

```sql
-- アプリケーション用ユーザーの作成
CREATE USER app_user WITH PASSWORD 'secure_password';

-- 必要なテーブルへの読み書き権限のみ付与
GRANT SELECT, INSERT, UPDATE ON users TO app_user;
GRANT SELECT ON products TO app_user;

-- 管理操作の権限は付与しない
-- DROP TABLE, CREATE TABLE, GRANT等はできない
```

## アクセス制御

### ユーザーの管理

```sql
-- ユーザーの作成
CREATE USER analyst WITH PASSWORD 'strong_password_here';

-- ロール（グループ）の作成
CREATE ROLE readonly;
CREATE ROLE readwrite;

-- ロールに権限を付与
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;

-- ユーザーにロールを割り当て
GRANT readonly TO analyst;
```

### スキーマレベルのアクセス制御

```sql
-- スキーマの作成
CREATE SCHEMA sensitive_data;

-- 特定ユーザーのみアクセス可能にする
REVOKE ALL ON SCHEMA sensitive_data FROM PUBLIC;
GRANT USAGE ON SCHEMA sensitive_data TO admin_user;
```

### 行レベルセキュリティ（PostgreSQL）

```sql
-- 行レベルセキュリティの有効化
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- ポリシーの作成：自部署のデータのみ参照可能
CREATE POLICY dept_policy ON employees
    FOR SELECT
    USING (department = current_setting('app.current_department'));

-- 設定例
SET app.current_department = '開発部';
SELECT * FROM employees;  -- 開発部のデータのみ返される
```

## データの暗号化

### パスワードのハッシュ化

```sql
-- ❌ 平文でパスワードを保存
INSERT INTO users (username, password)
VALUES ('tanaka', 'mypassword123');

-- ✅ ハッシュ化して保存（PostgreSQLのpgcrypto拡張）
CREATE EXTENSION pgcrypto;

INSERT INTO users (username, password_hash)
VALUES ('tanaka', crypt('mypassword123', gen_salt('bf')));

-- 認証時の照合
SELECT * FROM users
WHERE username = 'tanaka'
  AND password_hash = crypt('mypassword123', password_hash);
```

### 機密データの暗号化

```sql
-- PostgreSQL pgcrypto
-- 暗号化して保存
INSERT INTO credit_cards (user_id, encrypted_number)
VALUES (1, pgp_sym_encrypt('4111111111111111', 'encryption_key'));

-- 復号化して取得
SELECT pgp_sym_decrypt(encrypted_number, 'encryption_key')
FROM credit_cards WHERE user_id = 1;
```

## バックアップと復旧

### 定期バックアップ

```bash
# PostgreSQL
pg_dump -U postgres -d mydb -F c -f backup_$(date +%Y%m%d).dump

# MySQL
mysqldump -u root -p mydb > backup_$(date +%Y%m%d).sql
```

### リストア

```bash
# PostgreSQL
pg_restore -U postgres -d mydb backup_20240115.dump

# MySQL
mysql -u root -p mydb < backup_20240115.sql
```

### ポイントインタイムリカバリ（PITR）

特定の時点の状態にデータベースを復元する仕組みです。本番環境では設定を推奨します。

## SQLのベストプラクティス

### 1. 命名規則を統一する

```sql
-- ✅ 良い命名
CREATE TABLE order_items (
    order_item_id INTEGER PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ❌ 悪い命名
CREATE TABLE tbl_OrdItm (
    ID INT,
    oid INT,
    pid INT,
    qty INT,
    price FLOAT,
    dt DATETIME
);
```

### 2. 適切なデータ型を選ぶ

```sql
-- ❌ 数値にVARCHARを使う
CREATE TABLE products (
    price VARCHAR(20)     -- 比較や計算が正しく行えない
);

-- ✅ 適切なデータ型
CREATE TABLE products (
    price DECIMAL(10, 2)  -- 金額は固定小数点
);
```

### 3. NULLを適切に扱う

```sql
-- NULLになりうるカラムは明示的に処理する
SELECT
    name,
    COALESCE(department, '未配属') AS department,
    COALESCE(salary, 0) AS salary
FROM employees;
```

### 4. トランザクションを適切に使う

```sql
-- 関連する操作はトランザクションでまとめる
BEGIN;

INSERT INTO orders (customer_id, total_amount)
VALUES (1, 15000);

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (LASTVAL(), 101, 3, 5000);

UPDATE products SET stock = stock - 3 WHERE product_id = 101;

COMMIT;
```

### 5. マイグレーションを管理する

スキーマの変更はバージョン管理し、段階的に適用します。

```sql
-- マイグレーションファイル: V001_create_users.sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(200) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- マイグレーションファイル: V002_add_users_phone.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
```

### 6. 本番環境での安全な操作

```sql
-- ❌ いきなりALTER TABLE
ALTER TABLE users DROP COLUMN email;

-- ✅ 段階的に行う
-- 1. アプリケーションからemailカラムの使用を停止
-- 2. バックアップを取得
-- 3. emailカラムを削除
-- 4. 動作確認
```

### 7. クエリにコメントを書く

```sql
-- 月次売上レポート用クエリ
-- 当月の確定注文のみを対象とする
SELECT
    p.category,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.status = 'delivered'
  AND o.order_date >= DATE_TRUNC('month', CURRENT_DATE)
  AND o.order_date < DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month'
GROUP BY p.category
ORDER BY total_revenue DESC;
```

### 8. モニタリングとアラート

```sql
-- 長時間実行中のクエリを確認（PostgreSQL）
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - pg_stat_activity.query_start > INTERVAL '5 minutes';

-- ロックの確認（PostgreSQL）
SELECT
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks bk ON bk.locktype = bl.locktype
    AND bk.relation = bl.relation
    AND bk.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = bk.pid
WHERE NOT bl.granted;
```

## セキュリティチェックリスト

- [ ] パラメータ化クエリを使用しているか
- [ ] アプリケーションDBユーザーの権限は最小限か
- [ ] パスワードはハッシュ化されているか
- [ ] 機密データは暗号化されているか
- [ ] 定期バックアップは実施しているか
- [ ] 不要なユーザーアカウントは削除されているか
- [ ] データベースのバージョンは最新か
- [ ] 接続は暗号化（SSL/TLS）されているか
- [ ] 監査ログは有効か
- [ ] ネットワークアクセスは制限されているか

## シリーズ全体のまとめ

このシリーズで学んだ内容の全体像です。

### 基礎（第1〜5回）

| 回 | トピック | キーワード |
|----|---------|-----------|
| 1 | SQL入門 | RDBMS, テーブル, 行, 列 |
| 2 | SELECT | AS, DISTINCT, CASE, CAST |
| 3 | WHERE | 比較演算子, AND/OR, LIKE, IN, BETWEEN |
| 4 | ORDER BY/LIMIT | ASC, DESC, OFFSET |
| 5 | 集約関数 | COUNT, SUM, AVG, MAX, MIN |

### データ操作（第6〜11回）

| 回 | トピック | キーワード |
|----|---------|-----------|
| 6 | DML | INSERT, UPDATE, DELETE, UPSERT |
| 7 | INNER JOIN | テーブル結合, 自己結合 |
| 8 | OUTER JOIN | LEFT, RIGHT, FULL JOIN |
| 9 | サブクエリ | IN, EXISTS, CTE, 再帰CTE |
| 10 | GROUP BY | HAVING, ROLLUP, CUBE |
| 11 | UNION | INTERSECT, EXCEPT |

### DDLと高度な機能（第12〜16回）

| 回 | トピック | キーワード |
|----|---------|-----------|
| 12 | CREATE TABLE | データ型, ALTER TABLE |
| 13 | 制約 | PK, FK, UNIQUE, CHECK, NOT NULL |
| 14 | インデックス | B-Tree, EXPLAIN, カバリングインデックス |
| 15 | ビュー | VIEW, マテリアライズドビュー |
| 16 | トランザクション | ACID, 分離レベル, ロック |

### 実践と応用（第17〜20回）

| 回 | トピック | キーワード |
|----|---------|-----------|
| 17 | ウィンドウ関数 | ROW_NUMBER, RANK, LAG/LEAD |
| 18 | ストアドプロシージャ | 関数, トリガー, PL/pgSQL |
| 19 | パフォーマンス | EXPLAIN ANALYZE, 最適化 |
| 20 | セキュリティ | SQLインジェクション, アクセス制御 |

## 次のステップ

このシリーズを通じてSQLの基礎から応用まで学びました。さらにスキルを伸ばすために：

1. **実際のデータで練習する** — 公開データセットを使って分析
2. **特定のRDBMSを深く学ぶ** — PostgreSQL, MySQL等の固有機能
3. **ORMを学ぶ** — SQLAlchemy, Prisma, ActiveRecord等
4. **データベース設計を学ぶ** — 正規化理論、ER図
5. **大規模データの技術を学ぶ** — レプリケーション、シャーディング

SQLは現代のソフトウェア開発において必須のスキルです。ぜひ実践の中で腕を磨いてください。

---

お読みいただきありがとうございました。シリーズ全20回が皆様のSQL学習の一助となれば幸いです。
