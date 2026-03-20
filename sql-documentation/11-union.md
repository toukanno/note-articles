# 【SQL入門】第11回：UNION — 複数のクエリ結果を結合する

## はじめに

今回は、複数のSELECT文の結果を縦方向に結合する**集合演算子**を学びます。JOINが横方向の結合（テーブルの横につなぐ）なのに対し、UNIONは**縦方向の結合（結果を上下につなぐ）**です。

## 集合演算子の種類

| 演算子 | 説明 |
|--------|------|
| `UNION` | 和集合（重複を排除） |
| `UNION ALL` | 和集合（重複を含む） |
| `INTERSECT` | 積集合（共通する行のみ） |
| `EXCEPT` | 差集合（左にあって右にない行） |

## UNION：重複を排除して結合

```sql
SELECT カラム名 FROM テーブル1
UNION
SELECT カラム名 FROM テーブル2;
```

### 基本例

```sql
-- 営業部の社員名と人事部の社員名を結合
SELECT name FROM employees WHERE department = '営業部'
UNION
SELECT name FROM employees WHERE department = '人事部';
```

結果：

```
name
----------
田中太郎
伊藤由美
佐藤次郎
中村さくら
```

### UNIONのルール

1. **SELECT文のカラム数が同じ**であること
2. **対応するカラムのデータ型が互換性を持つ**こと
3. 結果のカラム名は**最初のSELECT文**のものが使われる

```sql
-- ✅ カラム数が同じ
SELECT name, salary FROM employees WHERE department = '営業部'
UNION
SELECT name, salary FROM employees WHERE department = '開発部';

-- ❌ カラム数が異なる → エラー
SELECT name, salary FROM employees
UNION
SELECT name FROM employees;
```

## UNION ALL：重複を含めて結合

UNIONは重複行を排除しますが、**UNION ALL**は重複をそのまま保持します。

```sql
SELECT department FROM employees
UNION ALL
SELECT department FROM employees;
-- → 16行（8行 × 2）

SELECT department FROM employees
UNION
SELECT department FROM employees;
-- → 4行（重複排除後）
```

> 💡 **パフォーマンスのヒント**：重複排除が不要な場合は`UNION ALL`を使いましょう。`UNION`はソート処理が発生するため遅くなります。

## INTERSECT：積集合

両方のクエリ結果に共通する行だけを返します。

```sql
-- 給与45万円以上かつ年齢30歳以上の社員
SELECT name FROM employees WHERE salary >= 450000
INTERSECT
SELECT name FROM employees WHERE age >= 30;
```

結果：

```
name
--------
田中太郎
佐藤次郎
高橋一郎
渡辺健太
```

> 📝 MySQLは`INTERSECT`をサポートしていません（MySQL 8.0.31以降で対応）。代わりにINNER JOINやIN + サブクエリを使います。

## EXCEPT（MINUS）：差集合

左のクエリ結果から、右のクエリ結果に含まれる行を除外します。

```sql
-- 給与45万円以上だが年齢30歳未満の社員
SELECT name FROM employees WHERE salary >= 450000
EXCEPT
SELECT name FROM employees WHERE age >= 30;
```

結果：

```
name
--------
鈴木花子
山田美咲
```

> 📝 OracleではEXCEPTの代わりに`MINUS`を使います。

## 実践的な使用パターン

### パターン1：異なるテーブルの統合

```sql
-- 正社員とアルバイトの名簿を統合
SELECT name, '正社員' AS 区分 FROM full_time_employees
UNION ALL
SELECT name, 'アルバイト' AS 区分 FROM part_time_employees
ORDER BY name;
```

### パターン2：条件別のラベル付け

```sql
SELECT name, salary, '高給' AS 区分
FROM employees WHERE salary >= 500000
UNION ALL
SELECT name, salary, '標準' AS 区分
FROM employees WHERE salary >= 400000 AND salary < 500000
UNION ALL
SELECT name, salary, '低給' AS 区分
FROM employees WHERE salary < 400000
ORDER BY salary DESC;
```

### パターン3：存在チェック

```sql
-- テーブルAにあってテーブルBにないデータを見つける
SELECT product_id FROM products
EXCEPT
SELECT product_id FROM discontinued_products;
```

### パターン4：集計結果の結合

```sql
-- 部署ごとの集計と全体の集計を一度に取得
SELECT department AS 区分, COUNT(*) AS 人数, AVG(salary) AS 平均給与
FROM employees
GROUP BY department
UNION ALL
SELECT '全体合計', COUNT(*), AVG(salary)
FROM employees
ORDER BY 区分;
```

### パターン5：MySQLでFULL JOINを実現

```sql
SELECT e.name, d.dept_name
FROM emp e
LEFT JOIN departments d ON e.dept_id = d.dept_id
UNION
SELECT e.name, d.dept_name
FROM emp e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

## ORDER BYの位置

集合演算子を使う場合、ORDER BYは**最後のSELECT文の後**に1回だけ書きます。

```sql
-- ✅ 正しい：最後にORDER BY
SELECT name, salary FROM employees WHERE department = '営業部'
UNION ALL
SELECT name, salary FROM employees WHERE department = '開発部'
ORDER BY salary DESC;

-- ❌ エラー：途中にORDER BYは書けない
SELECT name, salary FROM employees WHERE department = '営業部'
ORDER BY salary DESC  -- ← ここには書けない
UNION ALL
SELECT name, salary FROM employees WHERE department = '開発部';
```

途中のクエリをソートしたい場合は、サブクエリを使います：

```sql
SELECT * FROM (
    SELECT name, salary FROM employees
    WHERE department = '営業部'
    ORDER BY salary DESC
    LIMIT 3
) top_sales
UNION ALL
SELECT * FROM (
    SELECT name, salary FROM employees
    WHERE department = '開発部'
    ORDER BY salary DESC
    LIMIT 3
) top_dev;
```

## パフォーマンスの考慮

| 方法 | 特徴 |
|------|------|
| `UNION` | 重複排除のためソートが発生（遅い） |
| `UNION ALL` | ソート不要（速い） |
| `OR条件` | 単一テーブルなら集合演算子より効率的な場合が多い |

```sql
-- 同じテーブル内なら、UNIONよりORやINが効率的
-- ❌ 非効率
SELECT name FROM employees WHERE department = '営業部'
UNION
SELECT name FROM employees WHERE department = '開発部';

-- ✅ 効率的
SELECT name FROM employees
WHERE department IN ('営業部', '開発部');
```

## 実践練習

```sql
-- 練習1：全部署の一覧（社員テーブルと部署テーブルから重複なし）
SELECT department AS dept FROM employees WHERE department IS NOT NULL
UNION
SELECT dept_name FROM departments;

-- 練習2：高給者と古参社員のリスト（重複あり）
SELECT name, '高給' AS reason FROM employees WHERE salary >= 500000
UNION ALL
SELECT name, '古参' AS reason FROM employees WHERE hire_date < '2019-01-01';

-- 練習3：営業部にいて開発部にいない年齢層
SELECT age FROM employees WHERE department = '営業部'
EXCEPT
SELECT age FROM employees WHERE department = '開発部';
```

## まとめ

| 演算子 | 重複 | 説明 | MySQL対応 |
|--------|------|------|----------|
| `UNION` | 排除 | 和集合 | ✅ |
| `UNION ALL` | 保持 | 和集合（高速） | ✅ |
| `INTERSECT` | 排除 | 積集合 | 8.0.31+ |
| `EXCEPT` | 排除 | 差集合 | 8.0.31+ |

次回は、**CREATE TABLE**でテーブル設計の基本を学びます。
