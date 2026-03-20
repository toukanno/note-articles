# 【SQL入門】第3回：WHERE句 — 条件を指定してデータを絞り込む

## はじめに

前回はSELECT文の基本を学びました。今回は**WHERE句**を使って、条件に合うデータだけを取り出す方法を詳しく解説します。

## WHERE句の基本構文

```sql
SELECT カラム名
FROM テーブル名
WHERE 条件式;
```

## 比較演算子

### 基本の比較演算子

| 演算子 | 意味 | 例 |
|--------|------|-----|
| `=` | 等しい | `age = 30` |
| `<>` or `!=` | 等しくない | `age <> 30` |
| `<` | より小さい | `age < 30` |
| `>` | より大きい | `age > 30` |
| `<=` | 以下 | `age <= 30` |
| `>=` | 以上 | `age >= 30` |

### 数値の比較

```sql
-- 年齢が30歳以上の社員
SELECT name, age FROM employees
WHERE age >= 30;
```

結果：

```
name     | age
---------+----
田中太郎 |  30
佐藤次郎 |  35
高橋一郎 |  32
渡辺健太 |  40
```

### 文字列の比較

```sql
-- 開発部の社員
SELECT name, department FROM employees
WHERE department = '開発部';
```

結果：

```
name     | department
---------+-----------
鈴木花子 | 開発部
山田美咲 | 開発部
渡辺健太 | 開発部
```

### 日付の比較

```sql
-- 2020年以降に入社した社員
SELECT name, hire_date FROM employees
WHERE hire_date >= '2020-01-01';
```

結果：

```
name       | hire_date
-----------+-----------
田中太郎   | 2020-04-01
鈴木花子   | 2022-04-01
山田美咲   | 2021-04-01
伊藤由美   | 2023-04-01
中村さくら | 2024-04-01
```

## 論理演算子

### AND：複数条件をすべて満たす

```sql
-- 開発部かつ給与50万円以上
SELECT name, department, salary
FROM employees
WHERE department = '開発部'
  AND salary >= 500000;
```

結果：

```
name     | department | salary
---------+------------+-------
鈴木花子 | 開発部     | 500000
山田美咲 | 開発部     | 520000
渡辺健太 | 開発部     | 600000
```

### OR：いずれかの条件を満たす

```sql
-- 営業部または人事部の社員
SELECT name, department
FROM employees
WHERE department = '営業部'
   OR department = '人事部';
```

結果：

```
name       | department
-----------+-----------
田中太郎   | 営業部
佐藤次郎   | 人事部
伊藤由美   | 営業部
中村さくら | 人事部
```

### NOT：条件の否定

```sql
-- 開発部以外の社員
SELECT name, department
FROM employees
WHERE NOT department = '開発部';
```

### ANDとORの組み合わせ

ANDはORより優先されます。意図通りに動作させるには**括弧**を使いましょう。

```sql
-- ❌ 意図しない結果になりがち
SELECT * FROM employees
WHERE department = '営業部' OR department = '開発部'
  AND salary >= 500000;
-- ↑ これは department='営業部' OR (department='開発部' AND salary>=500000) と解釈される

-- ✅ 括弧で明示する
SELECT * FROM employees
WHERE (department = '営業部' OR department = '開発部')
  AND salary >= 500000;
```

## BETWEEN：範囲指定

```sql
-- 給与が45万〜55万の社員
SELECT name, salary
FROM employees
WHERE salary BETWEEN 450000 AND 550000;
```

これは以下と同じ意味です：

```sql
SELECT name, salary
FROM employees
WHERE salary >= 450000 AND salary <= 550000;
```

> 📝 `BETWEEN`は**両端を含む**点に注意してください。

### NOT BETWEEN

```sql
-- 給与が45万〜55万の範囲外の社員
SELECT name, salary
FROM employees
WHERE salary NOT BETWEEN 450000 AND 550000;
```

## IN：リスト内の値に一致

```sql
-- 営業部、開発部、経理部のいずれかに所属する社員
SELECT name, department
FROM employees
WHERE department IN ('営業部', '開発部', '経理部');
```

これは以下と同じ意味です：

```sql
SELECT name, department
FROM employees
WHERE department = '営業部'
   OR department = '開発部'
   OR department = '経理部';
```

### NOT IN

```sql
-- 営業部と人事部以外の社員
SELECT name, department
FROM employees
WHERE department NOT IN ('営業部', '人事部');
```

> ⚠️ **注意**：`IN`のリストに`NULL`が含まれると、`NOT IN`は期待通りに動作しません。

```sql
-- ❌ NULLが含まれると結果が空になる場合がある
WHERE department NOT IN ('営業部', NULL);

-- ✅ NULLは別途IS NOT NULLで処理する
WHERE department NOT IN ('営業部')
  AND department IS NOT NULL;
```

## LIKE：パターンマッチング

文字列の部分一致検索を行います。

### ワイルドカード

| ワイルドカード | 意味 | 例 |
|---------------|------|-----|
| `%` | 0文字以上の任意の文字列 | `'田%'` → 田中、田村、田... |
| `_` | ちょうど1文字の任意の文字 | `'田_'` → 田中、田村（2文字のみ） |

### 前方一致

```sql
-- 名前が「田」で始まる社員
SELECT name FROM employees
WHERE name LIKE '田%';
```

結果：

```
name
--------
田中太郎
```

### 後方一致

```sql
-- 名前が「郎」で終わる社員
SELECT name FROM employees
WHERE name LIKE '%郎';
```

結果：

```
name
--------
田中太郎
佐藤次郎
高橋一郎
```

### 部分一致

```sql
-- 名前に「美」が含まれる社員
SELECT name FROM employees
WHERE name LIKE '%美%';
```

結果：

```
name
--------
山田美咲
伊藤由美
```

### アンダースコアによる文字数指定

```sql
-- 名前がちょうど4文字の社員
SELECT name FROM employees
WHERE name LIKE '____';

-- 「佐藤」で始まり、その後にちょうど2文字続く社員
SELECT name FROM employees
WHERE name LIKE '佐藤__';
```

### エスケープ文字

`%`や`_`自体を検索したい場合は、`ESCAPE`句を使います。

```sql
-- カラムに「100%」という文字列が含まれるレコードを検索
SELECT * FROM products
WHERE description LIKE '%100!%%' ESCAPE '!';
```

### NOT LIKE

```sql
-- 名前が「田」で始まらない社員
SELECT name FROM employees
WHERE name NOT LIKE '田%';
```

## IS NULL / IS NOT NULL

NULLの判定には `=` ではなく `IS NULL` を使います。

```sql
-- 部署が未設定の社員
SELECT name FROM employees
WHERE department IS NULL;

-- 部署が設定されている社員
SELECT name FROM employees
WHERE department IS NOT NULL;
```

> ⚠️ **重要**：`WHERE department = NULL` は常に結果が空になります。NULLは「不明」を意味するため、`NULL = NULL`も`FALSE`となります。

## EXISTS：存在確認

サブクエリの結果が存在するかどうかを確認します（詳細は第9回で解説）。

```sql
-- 注文が存在する商品を取得
SELECT * FROM products p
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.product_id = p.product_id
);
```

## 条件の評価順序

WHERE句の条件は以下の優先順位で評価されます：

1. `()` — 括弧
2. `NOT` — 否定
3. `AND` — 論理積
4. `OR` — 論理和

```sql
-- 優先順位の例
WHERE A OR B AND C
-- は以下と同じ
WHERE A OR (B AND C)

-- AまたはBのいずれかで、かつCを満たすなら括弧を使う
WHERE (A OR B) AND C
```

## 実践：複雑な条件の組み立て

### 例1：複数条件の組み合わせ

```sql
-- 開発部で給与50万円以上、または経理部の社員を取得
SELECT name, department, salary
FROM employees
WHERE (department = '開発部' AND salary >= 500000)
   OR department = '経理部';
```

### 例2：日付と数値の複合条件

```sql
-- 2020年以降入社で、30歳未満、給与45万円以上の社員
SELECT name, age, salary, hire_date
FROM employees
WHERE hire_date >= '2020-01-01'
  AND age < 30
  AND salary >= 450000;
```

### 例3：LIKEとINの組み合わせ

```sql
-- 名前に「藤」が含まれ、営業部か人事部に所属する社員
SELECT name, department
FROM employees
WHERE name LIKE '%藤%'
  AND department IN ('営業部', '人事部');
```

## パフォーマンスのヒント

- `LIKE '%文字列'`（前方ワイルドカード）はインデックスが効かないため低速
- `IN`の値が多すぎる場合は、一時テーブルやJOINの利用を検討
- `OR`を多用するとパフォーマンスが低下する場合がある → `IN`や`UNION`に置き換えを検討
- 関数をカラムに適用すると、インデックスが無効になる場合がある

```sql
-- ❌ インデックスが効かない
WHERE YEAR(hire_date) = 2020

-- ✅ インデックスが効く
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01'
```

## まとめ

| 機能 | 構文 | 例 |
|------|------|-----|
| 比較 | `=, <>, <, >, <=, >=` | `WHERE age >= 30` |
| AND | `AND` | `WHERE a = 1 AND b = 2` |
| OR | `OR` | `WHERE a = 1 OR b = 2` |
| NOT | `NOT` | `WHERE NOT a = 1` |
| 範囲 | `BETWEEN` | `WHERE age BETWEEN 25 AND 35` |
| リスト | `IN` | `WHERE dept IN ('A', 'B')` |
| パターン | `LIKE` | `WHERE name LIKE '田%'` |
| NULL判定 | `IS NULL` | `WHERE dept IS NULL` |

次回は、`ORDER BY`と`LIMIT`を使って、データの並び替えと取得件数の制限を学びます。
