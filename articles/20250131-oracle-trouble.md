---
title: "Oracle のバージョンアップで「ORA-01791: SELECT式が無効です」が発生した"
emoji: "⚠️"
type: "tech"
topics:
  - "oracle"
  - "rails"
published: true
published_at: "2025-01-31 17:51"
hugo_date: "2025-01-31T17:51:00+09:00"
---

## はじめに

現在のプロジェクトでは Ruby on Rails と Oracle Database を使用しています。開発環境を Oracle 19c Enterprise から Oracle 23ai Free に移行したところ、正常に動作していたアプリケーションで以下のエラーが発生するようになりました。

```
ActiveRecord::StatementInvalid: OCIError: ORA-01791: SELECT式が無効です。
```

原因を特定できましたが、珍しい事例だったため、共有します。

この記事のサンプルコードは説明のためのもので、実際のアプリケーションとは無関係です。また、一部のコードは動作検証を行っていないため、挙動が異なる可能性があります。

## エラーの発生箇所

エラーが発生したのは `eager_load` と `order` を組み合わせた部分でした。以下のコードで再現できます。

```ruby
  def find_user(user_id)
    User
      .eager_load(:pages)
      .order('pages.sort_number ASC')
      .find(user_id)
  end
```

このコードを実行すると、2 つの SQL が発行され、そのうちの 1 つでエラーが発生します。

```sql
SELECT
  DISTINCT "USERS"."ID",
  FIRST_VALUE(pages.sort_number) OVER(
    PARTITION BY USERS.ID
    ORDER BY
      pages.sort_number
  ) AS alias_0__
FROM
  "USERS"
  LEFT OUTER JOIN "PAGES" ON "PAGES"."USER_ID" = "USERS"."ID"
WHERE
  "USERS"."ID" = 1
ORDER BY
  PAGES.SORT_NUMBER ASC;
FETCH FIRST
  1 ROWS ONLY;
```

シンプルな Ruby コードに対して、生成される SQL は複雑になっています。

## より簡単な SQL でエラーを再現

元の SQL には `FIRST_VALUE` 関数や `LEFT JOIN` などが含まれていました。これらを削除してもエラーが再現するかを確認したところ、以下の SQL で発生することが分かりました。

```sql
CREATE TABLE tests (name VARCHAR(255) NOT NULL, sort_number INT);
INSERT INTO tests (name, sort_number) VALUES ('a', 3);
INSERT INTO tests (name, sort_number) VALUES ('a', 2);
INSERT INTO tests (name, sort_number) VALUES ('b', 1);

SELECT DISTINCT name FROM tests ORDER BY sort_number FETCH FIRST 1 ROWS ONLY;
```

最初の 4 行はデータのセットアップです。最後の SQL は Oracle 19 では成功しますが、Oracle 23 では失敗します。

## 原因

原因は一般的な SQL の誤りです。通常、`SELECT DISTINCT` を実行する際に `SELECT` 句に含まれていないカラムで `ORDER BY` することはできません。例えば、以下の SQL は多くのデータベースでエラーとなります。

```sql
SELECT DISTINCT name FROM tests ORDER BY sort_number;
```

しかし、Oracle 19 では `FETCH FIRST 1 ROWS ONLY` を付けることでこの制約を回避できていたようです。Oracle 23 ではこの動作が変更され、エラーとなるようになりました。

## 対策

`eager_load` を使用せず、シンプルなクエリに修正しました。

```ruby
  def load_records(user_id)
    @user = User.find(user_id)
    @pages = @user.pages.order(:sort_number)
  end
```

これにより、ActiveRecord は標準的な SQL のみを発行するようになります。

## おわりに

今回の問題は、Oracle のバージョンアップによる SQL の厳密化が原因でした。

データベースのバージョンアップ時には SQL の動作変更が起きることもしばしばあります。ただ、Oracle ではこういった仕様変更はあまりみられないので、こんなエラーが発生するのかと驚かされました。調査している間は、リリースノートもなかなか読みづらく、それらしい情報も見つからず、なかなか苦しい時間でした。私と同じ事例で悩んでいる人は、そう多くないと思うのですが、何かの役に立ちましたら幸いです。
