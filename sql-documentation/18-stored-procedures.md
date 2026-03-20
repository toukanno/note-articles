# 【SQL入門】第18回：ストアドプロシージャと関数

## はじめに

同じSQL処理を何度も実行する場合、**ストアドプロシージャ**や**関数**としてデータベースに保存しておくと便利です。今回はこれらの作成と使い方を解説します。

## ストアドプロシージャとは

データベースサーバー上に保存されるSQL手続きの集合です。名前をつけて呼び出すことができます。

### メリット

- **再利用性**：同じ処理を何度でも呼び出せる
- **パフォーマンス**：コンパイル済みなので高速
- **セキュリティ**：テーブルへの直接アクセスを制限できる
- **保守性**：ロジックの一元管理

## ストアドプロシージャの作成

### PostgreSQL（PL/pgSQL）

```sql
CREATE OR REPLACE PROCEDURE give_raise(
    target_dept VARCHAR,
    raise_percent NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE employees
    SET salary = salary * (1 + raise_percent / 100)
    WHERE department = target_dept;

    RAISE NOTICE '% の給与を %% アップしました', target_dept, raise_percent;
END;
$$;
```

呼び出し：

```sql
CALL give_raise('開発部', 5);
```

### MySQL

```sql
DELIMITER //

CREATE PROCEDURE give_raise(
    IN target_dept VARCHAR(50),
    IN raise_percent DECIMAL(5,2)
)
BEGIN
    UPDATE employees
    SET salary = salary * (1 + raise_percent / 100)
    WHERE department = target_dept;

    SELECT CONCAT(target_dept, 'の給与を', raise_percent, '%アップしました') AS message;
END //

DELIMITER ;
```

呼び出し：

```sql
CALL give_raise('開発部', 5);
```

## ユーザー定義関数

プロシージャと異なり、**値を返す**ことができ、SELECT文の中で使えます。

### PostgreSQL

```sql
-- 年収を計算する関数
CREATE OR REPLACE FUNCTION annual_salary(monthly_salary INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN monthly_salary * 12;
END;
$$;
```

使用例：

```sql
SELECT name, salary, annual_salary(salary) AS 年収
FROM employees;
```

### テーブルを返す関数

```sql
-- 指定部署の社員を返す関数
CREATE OR REPLACE FUNCTION get_dept_employees(dept_name VARCHAR)
RETURNS TABLE (
    emp_name VARCHAR,
    emp_salary INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT name, salary
    FROM employees
    WHERE department = dept_name
    ORDER BY salary DESC;
END;
$$;
```

使用例：

```sql
SELECT * FROM get_dept_employees('開発部');
```

### MySQL

```sql
DELIMITER //

CREATE FUNCTION annual_salary(monthly_salary INT)
RETURNS INT
DETERMINISTIC
BEGIN
    RETURN monthly_salary * 12;
END //

DELIMITER ;
```

## 変数と制御構文

### 変数宣言

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION example_variables()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    emp_count INTEGER;
    avg_sal NUMERIC;
    emp_name VARCHAR(100) := '田中太郎';
BEGIN
    SELECT COUNT(*), AVG(salary)
    INTO emp_count, avg_sal
    FROM employees;

    RAISE NOTICE '社員数: %, 平均給与: %', emp_count, avg_sal;
END;
$$;
```

### IF文

```sql
CREATE OR REPLACE FUNCTION salary_level(sal INTEGER)
RETURNS VARCHAR
LANGUAGE plpgsql
AS $$
BEGIN
    IF sal >= 550000 THEN
        RETURN 'S';
    ELSIF sal >= 500000 THEN
        RETURN 'A';
    ELSIF sal >= 450000 THEN
        RETURN 'B';
    ELSE
        RETURN 'C';
    END IF;
END;
$$;
```

### LOOP / WHILE / FOR

```sql
-- FORループ
CREATE OR REPLACE FUNCTION process_employees()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    emp RECORD;
BEGIN
    FOR emp IN SELECT * FROM employees ORDER BY emp_id
    LOOP
        RAISE NOTICE '処理中: % (給与: %)', emp.name, emp.salary;

        IF emp.salary < 400000 THEN
            UPDATE employees
            SET salary = 400000
            WHERE emp_id = emp.emp_id;
        END IF;
    END LOOP;
END;
$$;
```

### WHILEループ

```sql
CREATE OR REPLACE FUNCTION countdown(n INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    result TEXT := '';
    i INTEGER := n;
BEGIN
    WHILE i > 0 LOOP
        result := result || i::TEXT || ' ';
        i := i - 1;
    END LOOP;
    RETURN result;
END;
$$;

SELECT countdown(5);  -- '5 4 3 2 1 '
```

## 例外処理

```sql
CREATE OR REPLACE PROCEDURE safe_transfer(
    from_account VARCHAR,
    to_account VARCHAR,
    amount NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    current_balance NUMERIC;
BEGIN
    -- 残高確認
    SELECT balance INTO current_balance
    FROM accounts WHERE account_id = from_account
    FOR UPDATE;

    -- 残高不足チェック
    IF current_balance < amount THEN
        RAISE EXCEPTION '残高不足です。現在残高: %', current_balance;
    END IF;

    -- 振込実行
    UPDATE accounts SET balance = balance - amount
    WHERE account_id = from_account;

    UPDATE accounts SET balance = balance + amount
    WHERE account_id = to_account;

    RAISE NOTICE '振込完了: % → %, 金額: %', from_account, to_account, amount;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'エラー発生: %', SQLERRM;
        RAISE;  -- エラーを再スロー
END;
$$;
```

## トリガー

テーブルに対するINSERT/UPDATE/DELETE時に自動的に実行される処理です。

### PostgreSQL

```sql
-- 更新日時を自動設定するトリガー関数
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;

-- トリガーの作成
CREATE TRIGGER trg_employees_update
    BEFORE UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

### MySQL

```sql
DELIMITER //

CREATE TRIGGER trg_employees_update
    BEFORE UPDATE ON employees
    FOR EACH ROW
BEGIN
    SET NEW.updated_at = NOW();
END //

DELIMITER ;
```

### 監査ログトリガー

```sql
-- 監査ログテーブル
CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 監査トリガー関数
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data, changed_by)
        VALUES (TG_TABLE_NAME, 'INSERT', to_jsonb(NEW), current_user);
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_user);
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, changed_by)
        VALUES (TG_TABLE_NAME, 'DELETE', to_jsonb(OLD), current_user);
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_emp_audit
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger();
```

## プロシージャ vs 関数 vs トリガー

| 項目 | プロシージャ | 関数 | トリガー |
|------|-------------|------|---------|
| 呼び出し | CALL文 | SELECT文内 | 自動実行 |
| 戻り値 | なし（OUT引数は可） | あり | TRIGGER型 |
| トランザクション | 制御可能 | 不可（関数内） | 呼び出し元に従う |
| 用途 | 業務処理 | 値の計算 | データ変更の監視 |

## 管理コマンド

```sql
-- PostgreSQL：関数の一覧
SELECT routine_name, routine_type
FROM information_schema.routines
WHERE routine_schema = 'public';

-- PostgreSQL：関数の削除
DROP FUNCTION annual_salary(INTEGER);
DROP PROCEDURE give_raise(VARCHAR, NUMERIC);

-- PostgreSQL：トリガーの削除
DROP TRIGGER trg_employees_update ON employees;

-- MySQL：プロシージャの一覧
SHOW PROCEDURE STATUS WHERE Db = 'mydb';

-- MySQL：関数の一覧
SHOW FUNCTION STATUS WHERE Db = 'mydb';
```

## 実践練習

```sql
-- 練習1：税込み価格を返す関数
CREATE OR REPLACE FUNCTION tax_included(price NUMERIC, tax_rate NUMERIC DEFAULT 0.10)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN ROUND(price * (1 + tax_rate));
END;
$$;

SELECT tax_included(1000);       -- 1100
SELECT tax_included(1000, 0.08); -- 1080

-- 練習2：部署の統計を返す関数
CREATE OR REPLACE FUNCTION dept_stats(dept VARCHAR)
RETURNS TABLE (人数 BIGINT, 平均給与 NUMERIC, 最高給与 INTEGER)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT COUNT(*), ROUND(AVG(salary)), MAX(salary)
    FROM employees WHERE department = dept;
END;
$$;

SELECT * FROM dept_stats('開発部');
```

## まとめ

| 機能 | 用途 | 呼び出し |
|------|------|---------|
| ストアドプロシージャ | 業務処理の実行 | `CALL proc()` |
| ユーザー定義関数 | 値の計算 | `SELECT func()` |
| トリガー | 自動処理 | INSERT/UPDATE/DELETE時 |

次回は、**パフォーマンスチューニング**でSQLを速くする技術を学びます。
