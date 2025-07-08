---
title: "Rails の基本的開発ツールの紹介"
emoji: "⛏️"
type: "tech"
topics:
  - "rspec"
  - "rails"
published: true
published_at: "2025-04-14 15:00"
hugo_date: "2025-04-14T15:00:00+09:00"
---

## はじめに

この記事では、Ruby on Rails 開発で私が日常的に使っているツールをいくつかピックアップしました。Rails を触り始めたばかりの初心者の方にも、参考になる内容を目指しています。日々の開発で「これを知っていてよかった」と感じる基本的なツールを中心に紹介していきます。

## Git
Git はバージョン管理ツールです。コミット、プッシュなどの基本機能については省略します。ここでは、私がよく使うサブコマンドを紹介します。

### git grep
`git grep` は、リポジトリ内のファイルに含まれる文字列を高速に検索できるコマンドです。特にリファクタリングやキーワードの一括置換前の確認に役立ちます。

```
# spec/ 以下のファイルで hoge を検索
git grep hoge -- spec/

# spec/ 以外のファイルについて hoge を検索
git grep hoge -- :^spec/
```

オプションによって正規表現を使ったり、単語で検索したりすることも可能です。

* [リポジトリ内の探しものは「git grep」で効率的な検索テクニックを紹介
](https://envader.plus/article/386)
* [Git のさまざまなツール - 検索 | Git Documentation](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E6%A4%9C%E7%B4%A2)

### git log と git blame
コードを読んでいるときに、なぜそのような実装になっているのかわからない場合や、不具合が疑われる場合は以下のコマンドで履歴を調べます。例えば、 spec/rails_helper.rb が複雑になっていて「設定を変更したいが、それまでの経緯がわからない」といった場合には以下のコマンドが役に立ちます。

```
# ファイル spec/rails_helper.rb の変更履歴を表示
git log -p -- spec/rails_helper.rb

# ファイル spec/rails_helper.rb の変更者・コミットハッシュを表示
git blame -- spec/rails_helper.rb
```

さらに、コミットハッシュから GitHub 上で該当プルリクエストを調べると、コード変更の背景がよくわかります。

## Ruby on Rails のデバッグ

### rails console
`rails console` は Ruby on Rails のフレームワーク同梱の調査用コマンドです。インタラクティブにコードの試し打ちをすることができます。コードを読むだけではわからないことを試しながら確認できるため、重宝します。

```
bin/rails console
```

以下の例は pry を使っている場合の例です。

```ruby
# モデル Customer からレコードを1件取得
customer = Customer.last

# customer の持っているメソッドや定数を出力
ls customer

# customer の持っているメソッドのうち name が含まれるものだけ出力
ls customer --grep name

# customer のクラス定義ファイルを出力
$ customer

# コンソール上では 全ての Ruby スクリプトが実行可能です。
# 検証用メソッドを追加したり、オーバーライドすることもできます
def customer.to_s
  "my name is customer"
end

customer.to_s
```

テキストエディタで編集したコードをすぐ反映させるには reload! を使いましょう。

```ruby
# Customer クラスの do_something メソッドを作っているものとする
Customer.last.do_something

# エディタでコードを書き換えたらリロード
reload!

# 最新コードが反映されているので再実行
Customer.last.do_something
```

その他のコマンドは `help` コマンドやドキュメントを参照してください。

* [【Rails】 rails console(rails c)の便利な使い方とは？](https://pikawaka.com/rails/rails-console)
* [pry | GitHub](https://github.com/pry/pry)
* [irb | GitHub](https://github.com/ruby/irb)

### binding.pry と binding.irb
デバッグコマンド `binding.pry` はデータの状態を調べる上で非常に便利なツールです。実行中のプログラムを一時停止させて、rails console で調査した内容のほとんど全てを実行できます。使い方は rails console とほとんど同じなので省略しますが、代わりにいくつかの初歩的な記事を紹介します。参考にしてみてください。

- [binding.pryキホンのキ | SmartHR Tech Blog](https://tech.smarthr.jp/entry/2021/11/08/143649)
- [Ruby on Railsの開発時に知っておくと幸せになるbinding.pryのデバッグTips｜Offers Tech Blog](https://zenn.dev/overflow_offers/articles/20220602-binding-pry-debug-tips)

注意: `binding.pry` は便利ですが最新のローダー(zeitwerk)では正しく動作しません。最新のローダーを使う場合は `binding.irb` を使いましょう。操作方法はほとんど同じです。

### rails routes

大量のコントローラがあるとルーティングが複雑になり、コードから読み取るのが難解な場合があります。そのような場合はルーティング出力コマンドを利用しましょう。routes.rb を想定した通りに記述できているのか確認するのにも使えます。

```
bin/rails routes

# this/path/has/something/wrong が含まれる行を出力
# パスがわかっているが対応するコントローラがどれかよくわからない時に便利
bin/rails routes --grep this/path/has/something/wrong

# コントローラを指定して出力
# 作ったコントローラが正しくルーティングされてるかを確認したり、コントローラが使われてないことを確認するのに便利
bin/rails routes --controller SomethingController

# 縦長に出力
# パスやコントローラが長くなると読みにくくなるので、上記の検索で見辛かった時に組み合わせて使うと便利
bin/rails routes --expanded
```

## テスト(RSpec) のデバッグ

テスト実行時も binding.pry などのデバッグツールは利用できます。ここでは、テストコマンド rspec についての細かいオプションを紹介します。

```
# 行番号を指定して特定のテストのみ実行
rspec spec/hoge_spec.rb:33

# 最初に失敗したテストで即終了
rspec spec/hoge_spec.rb --fail-fast

# 上記テストで書き出された最新ログを見る
tail -n 100 log/test.log
```

エラーメッセージが全く理解できない場合は、ライブラリの使い方が間違っていることがあります。そのような場合は省略されているスタックトレースを追いかけると手掛かりになることがあります。

```
rspec spec/hoge_spec.rb --backtrace
```

## データベースクライアント

コードの調査の大半は rails console で可能ですが、以下のような場合はデータベースクライアントを使うのが便利です。

* テーブルの一覧を確認したいとき
* テーブルの定義やインデックスを確認したいとき
* 複雑な SQL を書いて動作確認したいとき

### CUI クライアント

以下のコマンドで CUI クライアントを起動できます。

```
bin/rails dbconsole
```

実行するとデータベースアダプタに応じたクライアントが動作します。例えば Oracle なら sqlplus が起動します。

```デモ
# sqlplus が起動し、パスワード入力を求められる
bin/rails dbconsole

# Oracle Database の SQL 実行例
SELECT USERNAME FROM DBA_USERS ORDER BY USERNAME;
```

### GUIクライアント

CUIクライアントは限られた機能しかサポートしておらず、やや使いにくい部分があります。特に、 sqlplus は履歴を参照できない、SQL の結果出力が見づらいなどの問題があります。GUIクライアントの利用をお勧めします。Oracle の場合は公式にサポートされている SQLDeveloper が便利です。インストール方法や使い方については、下記の記事が参考になります。

- [SQL Developerの使い方！GUIで便利にデータベースを操作する](https://style.potepan.com/articles/24899.html)

## 非同期処理のデバッグ

非同期処理は rails のサーバープロセスとは別のプロセスを起動するのが一般的です。しかし、sidekiq プロセスは rails の開発環境で使えるようなホットリロード機能を持っていません。そのため、コードの修正を加えたりデバッガを差し込むにはプロセスの再起動が必要となります。

このような手間を省きたい場合は非同期処理を同期処理に置き換えてしまえば簡単です。

```diff
- SomethingWorker.perform_later(arguments)
+ SomethingWorker.new.perform(arguments)
```

これで、sidekiq プロセスを使わずに rails のプロセスで実行できます。ホットリロードは有効になりますし、ログの出力先も一本化され見やすくなります。エラーのスタックトレースもわかりやすくなるはずです。このようなコードの置き換えが面倒であれば、development モード時はアダプタを sidekiq ではなく [sucker_punch](https://github.com/brandonhilkert/sucker_punch) に差し替えることで、似たような動作を実現できるはずです。

## 終わりに

今回は、Ruby on Rails の開発で私が日常的に使っているツールの中から、特に基礎的なツールについて紹介しました。これらのツールを活用することで、コードの理解が深まり、トラブルシューティングも効率的に行えるようになります。特に rails console や binding.pry を使ったインタラクティブな検証は、初心者からベテランまで重宝される技術です。自分の開発スタイルに合った使い方を探してみてください。
