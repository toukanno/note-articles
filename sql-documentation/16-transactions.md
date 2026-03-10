# 【SQL入門】第16回：トランザクション — データの整合性を保つ

## はじめに

銀行の振込処理で、送金元の残高は減ったのに送金先に入金されなかったら大問題です。**トランザクション**は、このような不完全な状態を防ぎ、データの整合性を保証する仕組みです。

## トランザクションとは

トランザクションは、**1つの論理的な作業単位として扱われる一連のSQL操作**です。すべてが成功するか、すべてが失敗するかのどちらかになります。

```sql
-- 振込処理の例
BEGIN;  -- トランザクション開始

-- 送金元から10万円を引く
UPDATE accounts SET balance = balance - 100000
WHERE account_id = 'A001';

-- 送金先に10万円を加える
UPDATE accounts SET balance = balance + 100000
WHERE account_id = 'B001';

COMMIT;  -- すべて成功 → 確定
```

もしエラーが発生した場合：

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100000
WHERE account_id = 'A001';

-- ここでエラーが発生！

ROLLBACK;  -- すべて取り消し（A001の残高も元に戻る）
```

## ACID特性

トランザクションが保証する4つの性質です。

### 1. Atomicity（原子性）

トランザクション内の操作は「**すべて成功**」か「**すべて失敗**」のいずれかになります。中途半端な状態は許されません。

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE account_id = 'A001';
UPDATE accounts SET balance = balance + 100000 WHERE account_id = 'B001';
-- 両方成功しなければ、両方取り消される
COMMIT;
```

### 2. Consistency（一貫性）

トランザクションの前後で、データベースは常に一貫性のある状態を保ちます。制約（CHECK, FOREIGN KEYなど）に違反する操作は拒否されます。

### 3. Isolation（分離性）

同時に実行されるトランザクション同士が互いに干渉しないことを保証します。

### 4. Durability（耐久性）

COMMITされたデータは、システム障害が発生しても失われません。

## 基本構文

### PostgreSQL / SQL標準

```sql
BEGIN;               -- トランザクション開始
-- SQL操作 ...
COMMIT;              -- 確定
-- または
ROLLBACK;            -- 取消
```

### MySQL

```sql
START TRANSACTION;   -- トランザクション開始
-- SQL操作 ...
COMMIT;
-- または
ROLLBACK;
```

### 自動コミット（Auto Commit）

多くのRDBMSでは、トランザクションを明示的に開始しない場合、各SQL文が自動的にコミットされます。

```sql
-- 自動コミットモードの確認（MySQL）
SELECT @@autocommit;  -- 1 = ON

-- 自動コミットの無効化
SET autocommit = 0;

-- PostgreSQLは常にトランザクション内で動作するが、
-- 明示的にBEGINしなければ各文が自動コミットされる
```

## SAVEPOINT：部分的なロールバック

トランザクション内の特定の地点に名前をつけ、そこまで戻ることができます。

```sql
BEGIN;

INSERT INTO orders (order_id, customer_id, total_amount)
VALUES (100, 1, 50000);

SAVEPOINT sp1;  -- セーブポイントを設定

INSERT INTO order_items (order_id, product_id, quantity)
VALUES (100, 1, 2);

-- order_itemsの挿入に問題があった場合
ROLLBACK TO SAVEPOINT sp1;  -- sp1まで戻る（ordersの挿入は残る）

-- 別のデータを挿入
INSERT INTO order_items (order_id, product_id, quantity)
VALUES (100, 2, 1);

COMMIT;  -- ordersと修正後のorder_itemsが確定
```

## 分離レベル（Isolation Level）

複数のトランザクションが同時に実行される場合に発生しうる問題と、それを防ぐ分離レベルがあります。

### 発生しうる問題

| 問題 | 説明 |
|------|------|
| ダーティリード | 他のトランザクションの未コミットデータを読める |
| ノンリピータブルリード | 同じクエリを2回実行すると結果が変わる |
| ファントムリード | 同じ条件で検索すると行数が変わる |

### 4つの分離レベル

| 分離レベル | ダーティリード | ノンリピータブルリード | ファントムリード |
|-----------|:---:|:---:|:---:|
| READ UNCOMMITTED | 発生する | 発生する | 発生する |
| READ COMMITTED | 防止 | 発生する | 発生する |
| REPEATABLE READ | 防止 | 防止 | 発生する |
| SERIALIZABLE | 防止 | 防止 | 防止 |

```sql
-- 分離レベルの設定
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- PostgreSQLのデフォルト：READ COMMITTED
-- MySQLのデフォルト：REPEATABLE READ
```

### 具体例：ダーティリード

```
トランザクションA                    トランザクションB
──────────────                    ──────────────
BEGIN;
UPDATE emp SET salary = 999999
  WHERE emp_id = 1;
                                  BEGIN;
                                  SELECT salary FROM emp
                                    WHERE emp_id = 1;
                                  → 999999 （未コミットの値を読んでしまう！）
ROLLBACK;
                                  -- 999999という値は存在しなかったことに...
```

### 具体例：ノンリピータブルリード

```
トランザクションA                    トランザクションB
──────────────                    ──────────────
BEGIN;
SELECT salary FROM emp
  WHERE emp_id = 1;
→ 450000
                                  BEGIN;
                                  UPDATE emp SET salary = 500000
                                    WHERE emp_id = 1;
                                  COMMIT;
SELECT salary FROM emp
  WHERE emp_id = 1;
→ 500000 （さっきと値が違う！）
```

## ロック（Lock）

トランザクションの分離性を実現するために、データベースはロック機構を使います。

### 行ロック

```sql
-- SELECT FOR UPDATE：取得した行をロック
BEGIN;
SELECT * FROM accounts
WHERE account_id = 'A001'
FOR UPDATE;  -- この行がロックされる

-- 他のトランザクションはこの行の更新を待つ
UPDATE accounts SET balance = balance - 100000
WHERE account_id = 'A001';

COMMIT;  -- ロック解放
```

### ロックの種類

| ロック | 説明 | 他の読取 | 他の書込 |
|--------|------|:------:|:------:|
| 共有ロック（FOR SHARE） | 読み取り用 | ✅ | ❌ |
| 排他ロック（FOR UPDATE） | 更新用 | RDBMS依存 | ❌ |

### デッドロック

2つのトランザクションが互いにロックを待ち合う状態です。

```
トランザクションA                    トランザクションB
──────────────                    ──────────────
BEGIN;                            BEGIN;
UPDATE accounts SET ...           UPDATE accounts SET ...
  WHERE id = 'A001';               WHERE id = 'B001';
  (A001をロック)                    (B001をロック)

UPDATE accounts SET ...           UPDATE accounts SET ...
  WHERE id = 'B001';               WHERE id = 'A001';
  (B001のロック待ち...)             (A001のロック待ち...)

  → デッドロック！
```

RDBMSはデッドロックを検出し、一方のトランザクションを自動的にロールバックします。

### デッドロックの防止策

- ロックの取得順序を統一する（例：ID昇順）
- トランザクションを短くする
- 必要なロックを最初にまとめて取得する

## 実践的なトランザクションパターン

### パターン1：口座振込

```sql
BEGIN;

-- 残高確認
SELECT balance FROM accounts WHERE account_id = 'A001' FOR UPDATE;
-- balance: 500000

-- 残高チェック
-- (アプリケーション側で balance >= 100000 を確認)

-- 振込実行
UPDATE accounts SET balance = balance - 100000 WHERE account_id = 'A001';
UPDATE accounts SET balance = balance + 100000 WHERE account_id = 'B001';

-- 取引履歴を記録
INSERT INTO transactions (from_account, to_account, amount, transaction_date)
VALUES ('A001', 'B001', 100000, CURRENT_TIMESTAMP);

COMMIT;
```

### パターン2：在庫管理

```sql
BEGIN;

-- 在庫をロック付きで確認
SELECT stock FROM products WHERE product_id = 101 FOR UPDATE;
-- stock: 5

-- 在庫チェック（アプリ側）
-- stock >= 注文数量 であること

-- 在庫を減らす
UPDATE products SET stock = stock - 2 WHERE product_id = 101;

-- 注文を作成
INSERT INTO orders (customer_id, product_id, quantity)
VALUES (1, 101, 2);

COMMIT;
```

### パターン3：エラーハンドリング（PostgreSQL PL/pgSQL）

```sql
DO $$
BEGIN
    -- トランザクション内の処理
    UPDATE accounts SET balance = balance - 100000 WHERE account_id = 'A001';
    UPDATE accounts SET balance = balance + 100000 WHERE account_id = 'B001';

EXCEPTION
    WHEN OTHERS THEN
        -- エラー発生時はロールバック
        RAISE NOTICE 'Error: %', SQLERRM;
        -- PL/pgSQL内では自動的にROLLBACKされる
END $$;
```

## 実践練習

```sql
-- 練習1：基本的なトランザクション
BEGIN;
UPDATE employees SET salary = salary + 10000 WHERE emp_id = 1;
SELECT name, salary FROM employees WHERE emp_id = 1;
-- 確認後
COMMIT;

-- 練習2：SAVEPOINTを使う
BEGIN;
INSERT INTO employees (emp_id, name, department, salary)
VALUES (50, 'テスト太郎', '営業部', 400000);
SAVEPOINT sp1;
UPDATE employees SET salary = -100 WHERE emp_id = 50; -- 無効な値
ROLLBACK TO sp1;
UPDATE employees SET salary = 420000 WHERE emp_id = 50;
COMMIT;

-- 練習3：FOR UPDATEでロック
BEGIN;
SELECT * FROM employees WHERE emp_id = 1 FOR UPDATE;
UPDATE employees SET salary = salary * 1.1 WHERE emp_id = 1;
COMMIT;
```

## まとめ

| 概念 | 説明 |
|------|------|
| BEGIN | トランザクション開始 |
| COMMIT | 変更を確定 |
| ROLLBACK | 変更を取消 |
| SAVEPOINT | 部分ロールバック地点 |
| ACID | 原子性、一貫性、分離性、耐久性 |
| 分離レベル | 同時実行の制御レベル |
| ロック | データの排他制御 |

次回は、高度な分析に使える**ウィンドウ関数**を学びます。
