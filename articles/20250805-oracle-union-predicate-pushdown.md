---
title: "Oracle OptimizerによるUNION句のPredicate Pushdown最適化"
emoji: "📘"
type: "tech"
topics:
  - "oracle"
  - "SQL"
published: true
published_at: "2025-08-05 17:40"
hugo_date: "2025-08-05T17:40:00+09:00"
---

## はじめに

私は SQL の UNION といえば「最適化が難しく、利用を避けるべきもの」として記憶していました。しかし、これは誤解でした。適切に利用すれば DB によって最適化され、効率的に動作するケースが存在します。Oracle 23.5 で実験した結果を紹介します。

## 実験

全く同じカラムで構成された2つのテーブルを UNION し、それを検索した時に実行計画がどうなるか観察してみましょう。

### データベースの作成

```
sqlplus SYS/${ORACLE_PWD}@FREEPDB1 AS SYSDBA
```

```sql
CREATE USER exp_user IDENTIFIED BY hoge;
GRANT CREATE SESSION TO exp_user;
GRANT CREATE TABLE TO exp_user;
GRANT UNLIMITED TABLESPACE TO exp_user;
GRANT CREATE VIEW TO exp_user;
```

### テーブルの作成

社員テーブルと、退職済み社員テーブルという2つのテーブルを作成します。

```
sqlplus exp_user/hoge@FREEPDB1
```

```sql
CREATE TABLE employees (
    employee_id   NUMBER(6)    PRIMARY KEY,
    employee_name VARCHAR2(100) NOT NULL,
    salary        NUMBER(8)
);

CREATE TABLE archived_employees (
    employee_id   NUMBER(6)    PRIMARY KEY,
    employee_name VARCHAR2(100) NOT NULL,
    salary        NUMBER(8)
);
```

### データの作成

```sql
INSERT INTO employees (employee_id, employee_name, salary) VALUES (101, '山田 太郎', 6000000);
INSERT INTO employees (employee_id, employee_name, salary) VALUES (102, '鈴木 花子', 7500000);
INSERT INTO employees (employee_id, employee_name, salary) VALUES (103, '佐藤 次郎', 5500000);
INSERT INTO employees (employee_id, employee_name, salary) VALUES (104, '高橋 三郎', 8000000);
INSERT INTO employees (employee_id, employee_name, salary) VALUES (105, '田中 良子', 5000000);

INSERT INTO archived_employees (employee_id, employee_name, salary) VALUES (201, '伊藤 博文', 7000000);
INSERT INTO archived_employees (employee_id, employee_name, salary) VALUES (202, '渡辺 直美', 6500000);
INSERT INTO archived_employees (employee_id, employee_name, salary) VALUES (104, '高橋 三郎', 8000000);

COMMIT;
```

### インデックスの定義

```sql
CREATE INDEX name_idx1 ON employees(employee_name);
CREATE INDEX name_idx2 ON archived_employees(employee_name);
```

### ビューの定義

```sql
CREATE OR REPLACE VIEW all_employees AS
SELECT
    employee_id,
    employee_name,
    salary
FROM
    employees
UNION ALL
SELECT
    employee_id,
    employee_name,
    salary
FROM
    archived_employees;
```

### 実行計画の確認

```sql
EXPLAIN PLAN FOR SELECT * FROM all_employees WHERE employee_name = '高橋 三郎'

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

出力結果は下記の通りでした。

```
------------------------------------------------------------------------------------------------------------
| Id  | Operation                             | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                      |                    |     2 |   190 |     4   (0)| 00:00:01 |
|   1 |  VIEW                                 | ALL_EMPLOYEES      |     2 |   190 |     4   (0)| 00:00:01 |
|   2 |   UNION-ALL                           |                    |     2 |   174 |     4   (0)| 00:00:01 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES          |     1 |    87 |     2   (0)| 00:00:01 |
|*  4 |     INDEX RANGE SCAN                  | NAME_IDX1          |     1 |       |     1   (0)| 00:00:01 |
|   5 |    TABLE ACCESS BY INDEX ROWID BATCHED| ARCHIVED_EMPLOYEES |     1 |    87 |     2   (0)| 00:00:01 |
|*  6 |     INDEX RANGE SCAN                  | NAME_IDX2          |     1 |       |     1   (0)| 00:00:01 |
------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   4 - access("EMPLOYEE_NAME"='高橋 三郎')
   6 - access("EMPLOYEE_NAME"='高橋 三郎')
```

ポイントは 4行目、6行目と `Predicate Information` です。これらの行は Oracle のオプティマイザが `UNION` 結果に対して `WHERE` 句をかけるのではなく、`UNION` 前の2つのテーブルに対して個別に `WHERE` 句をかけ、その結果を返していることを示しています。インデックスもしっかりと適用されているので、高速に動作するはずです。

ちなみに、インデックスがない場合の実行計画は下記の通りです。

```
------------------------------------------------------------------------------------------
| Id  | Operation           | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                    |     2 |   190 |     6   (0)| 00:00:01 |
|   1 |  VIEW               | ALL_EMPLOYEES      |     2 |   190 |     6   (0)| 00:00:01 |
|   2 |   UNION-ALL         |                    |     2 |   174 |     6   (0)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL| EMPLOYEES          |     1 |    87 |     3   (0)| 00:00:01 |
|*  4 |    TABLE ACCESS FULL| ARCHIVED_EMPLOYEES |     1 |    87 |     3   (0)| 00:00:01 |
------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - filter("EMPLOYEE_NAME"='高橋 三郎')
   4 - filter("EMPLOYEE_NAME"='高橋 三郎')
```

この場合でも `Predicate Information` を読むと、`UNION` 前の2つのテーブルに対して個別に `WHERE` 句をかけ、その結果を返していることがわかります。

## おわりに


実際にテーブルを作成して `UNION` 句を使ってもインデックスが利用されることを確認しました。今回紹介した最適化は、一般には [Predicate Pushdown](https://www.dremio.com/wiki/predicate-pushdown/) と呼ばれているようです。これで、安心して `UNION` を使うことができますね。アプリケーションの複雑さをデータベースで吸収するのに役立つはずです。

今回は前置きなしに `UNION ALL` を利用しましたが、 `ALL` を与えない場合にはパフォーマンスの差があります。今回のテストデータではほとんど差がなかったため省略しましたが、実際に利用するときは注意してください。

また、今回実験したテーブルはうまくいく前提のシンプルな設計でした。そのため、より複雑なテーブルを取り扱う場合には、期待通り動かない可能性もあります。実行計画を確認した方が良いでしょう。
