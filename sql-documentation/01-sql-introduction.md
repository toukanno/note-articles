# 【SQL入門】第1回：SQLとは何か？データベースの基礎を理解しよう

## はじめに

この記事は、SQLを初めて学ぶ方に向けた全20回のシリーズの第1回です。SQLの基本概念からデータベースの仕組みまで、丁寧に解説していきます。

## SQLとは？

**SQL（Structured Query Language）** は、リレーショナルデータベース管理システム（RDBMS）を操作するための標準的な言語です。1970年代にIBMで開発され、現在ではデータベースを扱うあらゆる場面で使われています。

SQLを使うと、以下のようなことができます：

- データの検索（取得）
- データの追加・更新・削除
- テーブルやデータベースの作成・変更
- アクセス権限の管理

## リレーショナルデータベースとは

リレーショナルデータベース（RDB）は、データを**テーブル（表）**の形式で管理するデータベースです。

### テーブルの構造

```
┌──────────────────────────────────────────────┐
│                 employees テーブル              │
├────────┬──────────┬──────┬───────────────────┤
│ emp_id │ name     │ age  │ department        │
├────────┼──────────┼──────┼───────────────────┤
│ 1      │ 田中太郎 │ 30   │ 営業部            │
│ 2      │ 鈴木花子 │ 25   │ 開発部            │
│ 3      │ 佐藤次郎 │ 35   │ 人事部            │
│ 4      │ 山田美咲 │ 28   │ 開発部            │
└────────┴──────────┴──────┴───────────────────┘
```

- **行（レコード / Row）**：1件のデータを表す（横方向）
- **列（カラム / Column）**：データの属性を表す（縦方向）
- **主キー（Primary Key）**：各レコードを一意に識別する列（上の例では `emp_id`）

## 代表的なRDBMS

| RDBMS | 特徴 | 用途 |
|-------|------|------|
| **MySQL** | オープンソース、高速 | Webアプリケーション全般 |
| **PostgreSQL** | 高機能、標準準拠 | 大規模システム、地理情報 |
| **SQLite** | 軽量、ファイルベース | モバイルアプリ、組み込み |
| **Oracle Database** | 商用、高信頼性 | 企業の基幹システム |
| **SQL Server** | Microsoft製、.NET連携 | Windows環境のシステム |

## SQLの分類

SQLの命令は、大きく4つに分類されます。

### 1. DDL（Data Definition Language）- データ定義言語

データベースやテーブルの構造を定義する命令です。

```sql
-- テーブルの作成
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    department VARCHAR(50)
);

-- テーブルの削除
DROP TABLE employees;

-- テーブル構造の変更
ALTER TABLE employees ADD COLUMN email VARCHAR(200);
```

### 2. DML（Data Manipulation Language）- データ操作言語

データの検索・追加・更新・削除を行う命令です。

```sql
-- データの検索
SELECT * FROM employees;

-- データの追加
INSERT INTO employees (emp_id, name, age, department)
VALUES (5, '高橋一郎', 32, '経理部');

-- データの更新
UPDATE employees SET age = 31 WHERE emp_id = 1;

-- データの削除
DELETE FROM employees WHERE emp_id = 3;
```

### 3. DCL（Data Control Language）- データ制御言語

アクセス権限を制御する命令です。

```sql
-- 権限の付与
GRANT SELECT ON employees TO user_name;

-- 権限の取消
REVOKE SELECT ON employees FROM user_name;
```

### 4. TCL（Transaction Control Language）- トランザクション制御言語

トランザクション（一連の処理のまとまり）を制御する命令です。

```sql
-- トランザクション開始
BEGIN TRANSACTION;

-- 確定
COMMIT;

-- 取消
ROLLBACK;
```

## SQLの基本ルール

SQLを書く際の基本的なルールを押さえておきましょう。

### 1. 大文字・小文字の区別

```sql
-- 以下はすべて同じ意味
SELECT * FROM employees;
select * from employees;
Select * From Employees;
```

SQL命令（キーワード）は大文字でも小文字でも動作しますが、**キーワードは大文字、テーブル名・カラム名は小文字**にするのが一般的な慣習です。

### 2. 文の終端

SQL文はセミコロン（`;`）で終わります。

```sql
SELECT * FROM employees;
SELECT name FROM employees;
```

### 3. コメント

```sql
-- 1行コメント（ハイフン2つ）

/*
  複数行コメント
  この部分は実行されません
*/

SELECT name /* インラインコメントも可能 */ FROM employees;
```

### 4. 文字列はシングルクォート

```sql
-- 文字列はシングルクォートで囲む
SELECT * FROM employees WHERE name = '田中太郎';

-- ダブルクォートはカラム名やテーブル名に使う（RDBMS依存）
SELECT "name" FROM employees;
```

## 最初のSQL：SELECT文

データベースの内容を見るための最も基本的なSQL文です。

```sql
-- テーブルの全データを取得
SELECT * FROM employees;
```

- `SELECT`：「取得せよ」という命令
- `*`：すべての列を対象とする
- `FROM employees`：`employees`テーブルから

実行結果：

```
emp_id | name     | age | department
-------+----------+-----+-----------
     1 | 田中太郎 |  30 | 営業部
     2 | 鈴木花子 |  25 | 開発部
     3 | 佐藤次郎 |  35 | 人事部
     4 | 山田美咲 |  28 | 開発部
```

## 学習環境のセットアップ

SQLを実際に試すには、以下の方法がおすすめです。

### 方法1：オンラインSQL実行環境

ブラウザ上でSQLを試すことができるサービスがあります。インストール不要で手軽に始められます。

### 方法2：SQLiteを使う

```bash
# macOSの場合（プリインストール済み）
sqlite3 practice.db

# Ubuntuの場合
sudo apt install sqlite3
sqlite3 practice.db
```

### 方法3：MySQLやPostgreSQLをインストール

本格的に学びたい場合は、MySQLやPostgreSQLをローカルにインストールして使うことをおすすめします。Dockerを使えば手軽に環境を作れます。

```bash
# Docker でMySQL環境を立ち上げる例
docker run --name mysql-practice -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql:8
```

## 練習用データベースの作成

これからのシリーズで使う練習用データを作りましょう。

```sql
-- 社員テーブル
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INTEGER,
    department VARCHAR(50),
    salary INTEGER,
    hire_date DATE
);

-- サンプルデータの投入
INSERT INTO employees VALUES (1, '田中太郎', 30, '営業部', 450000, '2020-04-01');
INSERT INTO employees VALUES (2, '鈴木花子', 25, '開発部', 500000, '2022-04-01');
INSERT INTO employees VALUES (3, '佐藤次郎', 35, '人事部', 480000, '2018-04-01');
INSERT INTO employees VALUES (4, '山田美咲', 28, '開発部', 520000, '2021-04-01');
INSERT INTO employees VALUES (5, '高橋一郎', 32, '経理部', 470000, '2019-04-01');
INSERT INTO employees VALUES (6, '伊藤由美', 27, '営業部', 440000, '2023-04-01');
INSERT INTO employees VALUES (7, '渡辺健太', 40, '開発部', 600000, '2015-04-01');
INSERT INTO employees VALUES (8, '中村さくら', 23, '人事部', 380000, '2024-04-01');
```

## まとめ

この記事では以下のことを学びました：

- **SQL**はリレーショナルデータベースを操作するための標準言語
- データは**テーブル（行と列）**の形式で管理される
- SQLは**DDL、DML、DCL、TCL**の4種類に分類される
- 基本ルール：セミコロンで文を終え、文字列はシングルクォートで囲む
- `SELECT * FROM テーブル名` で全データを取得できる

次回は、SELECT文をより詳しく学び、必要なデータだけを取り出す方法を解説します。

---

📝 **シリーズ目次**
1. **SQLとは何か？データベースの基礎を理解しよう** ← 今回
2. SELECT文の基本：必要なデータを取り出そう
3. WHERE句：条件を指定してデータを絞り込む
4. ORDER BYとLIMIT：データの並び替えと件数制限
5. 集約関数：COUNT, SUM, AVG, MAX, MIN
6. INSERT, UPDATE, DELETE：データの追加・更新・削除
7. JOIN入門：テーブルを結合しよう（INNER JOIN）
8. JOIN応用：LEFT JOIN, RIGHT JOIN, FULL JOIN
9. サブクエリ：SQLの中にSQLを書く
10. GROUP BYとHAVING：データをグループ化する
11. UNION：複数のクエリ結果を結合する
12. CREATE TABLE：テーブル設計の基本
13. 制約：NOT NULL, UNIQUE, CHECK, FOREIGN KEY
14. インデックス：検索を高速化する仕組み
15. ビュー：仮想テーブルを活用する
16. トランザクション：データの整合性を保つ
17. ウィンドウ関数：高度な分析クエリ
18. ストアドプロシージャと関数
19. パフォーマンスチューニング：SQLを速くする技術
20. セキュリティとベストプラクティス
