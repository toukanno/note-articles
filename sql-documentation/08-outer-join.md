# 【SQL入門】第8回：JOIN応用 — LEFT JOIN, RIGHT JOIN, FULL JOIN

## はじめに

前回はINNER JOINで「両テーブルに一致するデータ」を取得する方法を学びました。今回は**外部結合（Outer Join）**を使って、「一致しないデータ」も含めて取得する方法を解説します。

## 外部結合の種類

| 結合タイプ | 説明 |
|-----------|------|
| LEFT JOIN | 左テーブルの全行 + 右テーブルの一致行 |
| RIGHT JOIN | 右テーブルの全行 + 左テーブルの一致行 |
| FULL JOIN | 両テーブルの全行 |

## LEFT JOIN（LEFT OUTER JOIN）

**左側のテーブルの全行**を保持し、右側のテーブルに一致するものがなければNULLで埋めます。

```sql
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署名
FROM emp e
LEFT JOIN departments d
    ON e.dept_id = d.dept_id;
```

結果：

```
社員名     | 部署名
-----------+--------
田中太郎   | 営業部
鈴木花子   | 開発部
佐藤次郎   | 人事部
山田美咲   | 開発部
高橋一郎   | 経理部
伊藤由美   | 営業部
渡辺健太   | 開発部
中村さくら | NULL     ← dept_idがNULLだが、社員は表示される
```

> 📝 INNER JOINでは消えていた中村さくらが、LEFT JOINでは表示されます。一致する部署がないため、部署名はNULLになります。

### LEFT JOINの図解

```
     emp (左)          departments (右)
    ┌─────────┐        ┌──────────┐
    │ ████████████████████████████ │  ← 一致する行
    │ █████████│        │          │
    │ █████████│        │          │  ← 左テーブルにしかない行（NULLで埋まる）
    └─────────┘        │          │
                       │          │  ← 右テーブルにしかない行（表示されない）
                       └──────────┘
```

## RIGHT JOIN（RIGHT OUTER JOIN）

**右側のテーブルの全行**を保持します。

```sql
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署名
FROM emp e
RIGHT JOIN departments d
    ON e.dept_id = d.dept_id;
```

結果：

```
社員名     | 部署名
-----------+--------
田中太郎   | 営業部
伊藤由美   | 営業部
鈴木花子   | 開発部
山田美咲   | 開発部
渡辺健太   | 開発部
佐藤次郎   | 人事部
高橋一郎   | 経理部
NULL       | 広報部   ← 社員がいない部署も表示される
```

> 💡 **実務では LEFT JOIN を使うことが圧倒的に多い**です。RIGHT JOINはテーブルの順番を入れ替えてLEFT JOINに書き換えられるためです。

```sql
-- RIGHT JOIN
SELECT e.name, d.dept_name
FROM emp e RIGHT JOIN departments d ON e.dept_id = d.dept_id;

-- 同じ結果をLEFT JOINで（テーブル順を逆にする）
SELECT e.name, d.dept_name
FROM departments d LEFT JOIN emp e ON d.dept_id = e.dept_id;
```

## FULL JOIN（FULL OUTER JOIN）

**両方のテーブルの全行**を返します。

```sql
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署名
FROM emp e
FULL JOIN departments d
    ON e.dept_id = d.dept_id;
```

結果：

```
社員名     | 部署名
-----------+--------
田中太郎   | 営業部
鈴木花子   | 開発部
佐藤次郎   | 人事部
山田美咲   | 開発部
高橋一郎   | 経理部
伊藤由美   | 営業部
渡辺健太   | 開発部
中村さくら | NULL     ← 部署が未割当の社員
NULL       | 広報部   ← 社員がいない部署
```

> ⚠️ MySQLはFULL JOINをサポートしていません。代わりにLEFT JOINとRIGHT JOINをUNIONで組み合わせます。

```sql
-- MySQLでFULL JOINを実現
SELECT e.name, d.dept_name
FROM emp e LEFT JOIN departments d ON e.dept_id = d.dept_id
UNION
SELECT e.name, d.dept_name
FROM emp e RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

## JOINの比較まとめ

```
INNER JOIN:   一致する行のみ
              emp ∩ departments

LEFT JOIN:    左テーブルの全行 + 一致する右テーブル
              emp の全行 + departments の一致行

RIGHT JOIN:   右テーブルの全行 + 一致する左テーブル
              departments の全行 + emp の一致行

FULL JOIN:    両テーブルの全行
              emp ∪ departments
```

## 外部結合の実践パターン

### パターン1：一致しないデータだけを取得

```sql
-- 部署未配属の社員（LEFT JOINでNULLの行を抽出）
SELECT e.name
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

結果：

```
name
-----------
中村さくら
```

```sql
-- 社員が一人もいない部署
SELECT d.dept_name
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;
```

結果：

```
dept_name
---------
広報部
```

### パターン2：外部結合 + 集約関数

```sql
-- 全部署の社員数（社員がいない部署も0で表示）
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS member_count
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name
ORDER BY member_count DESC;
```

結果：

```
dept_name | member_count
----------+-------------
開発部    | 3
営業部    | 2
人事部    | 1
経理部    | 1
広報部    | 0
```

> ⚠️ `COUNT(*)`ではなく`COUNT(e.emp_id)`を使う点に注意。`COUNT(*)`だとNULLの行もカウントされ、広報部が1になってしまいます。

### パターン3：複数テーブルの外部結合

```sql
SELECT
    e.name AS 社員名,
    d.dept_name AS 部署,
    p.project_name AS プロジェクト
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id
LEFT JOIN project_members pm ON e.emp_id = pm.emp_id
LEFT JOIN projects p ON pm.project_id = p.project_id
ORDER BY e.name;
```

### パターン4：外部結合のWHERE句の注意点

```sql
-- ❌ 意図しない結果：LEFT JOINがINNER JOINのように動作する
SELECT e.name, d.dept_name
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = '東京';
-- dept_idがNULLの社員はd.locationもNULLとなりWHEREで除外される

-- ✅ ON句に条件を追加する
SELECT e.name, d.dept_name
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id AND d.location = '東京';
-- 左テーブルの全行が保持され、東京以外の部署はNULLになる
```

## NATURAL JOIN

同じカラム名を自動的に結合条件にする結合です。

```sql
-- dept_idが共通カラムの場合
SELECT * FROM emp NATURAL JOIN departments;
```

> ⚠️ **NATURAL JOINは非推奨**です。テーブル構造の変更で意図しない結合が発生する可能性があります。常にON句で結合条件を明示しましょう。

## USING句

ON句の簡略記法で、両テーブルに同じ名前のカラムがある場合に使えます。

```sql
-- USING句
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d USING (dept_id);

-- 上記は以下と同じ
SELECT e.name, d.dept_name
FROM emp e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

## 実践練習

```sql
-- 練習1：全社員の名前と部署名を表示（未配属も含む）
SELECT e.name, COALESCE(d.dept_name, '未配属') AS 部署
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id;

-- 練習2：社員がいない部署を見つける
SELECT d.dept_name
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;

-- 練習3：全部署の社員数と平均給与（0名の部署も表示）
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS 人数,
    COALESCE(AVG(e.salary), 0) AS 平均給与
FROM departments d
LEFT JOIN emp e ON d.dept_id = e.dept_id
GROUP BY d.dept_name;

-- 練習4：全社員と全部署を表示（FULL JOIN）
SELECT
    COALESCE(e.name, '(社員なし)') AS 社員,
    COALESCE(d.dept_name, '(未配属)') AS 部署
FROM emp e
FULL JOIN departments d ON e.dept_id = d.dept_id;
```

## まとめ

| 結合 | 左テーブル | 右テーブル | 使い道 |
|------|-----------|-----------|--------|
| INNER JOIN | 一致のみ | 一致のみ | 両方にあるデータ |
| LEFT JOIN | 全行 | 一致のみ+NULL | 左を基準にデータ取得 |
| RIGHT JOIN | 一致のみ+NULL | 全行 | 右を基準にデータ取得 |
| FULL JOIN | 全行 | 全行 | すべてのデータ |

次回は、SQLの中にSQLを書く**サブクエリ**を学びます。
