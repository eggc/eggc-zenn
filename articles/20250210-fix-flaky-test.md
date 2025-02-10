---
title: "RSpec における Flaky Test の修正方法"
emoji: "🔍"
type: "tech"
topics:
  - "rspec"
published: true
published_at: "2025-02-10 23:59"
hugo_date: "2025-02-10T23:59:00+09:00"
---

## はじめに

Flaky Test（不安定なテスト）とは、同じコードに対して実行するたびに結果が変わる可能性のあるテストを指します。リトライを繰り返していれば成功するので、見逃されてしまうことも多いと思います。再現させるのも難しいので、進んで調査する人もあまりいません。しかし、CI を回すのにもお金がかかりますし、待つ時間もコストです。素早く改善することが望ましいです。

この記事では、私が遭遇した中でも簡単な Flaky Test を取り上げて、その改善方法を紹介します。まだ Flaky Test の修正をしたことがない方の参考になるように、具体的なサンプルコードを示しながら、ゆっくり説明しています。

## 事例1：定数の再定義
### サンプルコード

以下は Flaky Test を含むコードの例です。二つのクラス Loader, Writer とそれぞれのテストがあります。Loader, Writer の機能はさほど重要ではないため、かなり簡略化しています。

```ruby
# lib/loader.rb
require "pathname"

class Loader
  BASE_PATH = "/base_path"

  def file_path(name)
    Pathname.new(BASE_PATH).join(name)
  end
end

# lib/writer.rb
class Writer
  def initialize(loader)
    @loader = loader
  end

  def write(content, filename)
    path = @loader.file_path(filename)
    File.write(path, content)
  end
end

# spec/loader_spec.rb
RSpec.describe Loader do
  describe '#file_path' do
    subject { Loader.new.file_path("backup.tmp") }

    it 'returns correct path' do
      is_expected.to eq(Pathname.new("/base_path/backup.tmp"))
    end
  end
end

# spec/writer_spec.rb
RSpec.describe Writer do
  before do
    Loader::BASE_PATH = "/tmp"  # 問題のコード
  end

  it 'writes to temporary directory' do
    writer = Writer.new(Loader.new)
    # テスト内容
  end
end
```

この時 `writer_spec.rb` と `loader_spec.rb` はそれぞれ別々に実行すると成功しますが、まとめて実行すると失敗します。

| 実行コマンド                                    | 結果     |
|-------------------------------------------------|----------|
| `rspec spec/loader_spec.rb`                     | 成功     |
| `rspec spec/writer_spec.rb`                     | 成功     |
| `rspec spec/loader_spec.rb spec/writer_spec.rb` | 成功     |
| `rspec spec/writer_spec.rb spec/loader_spec.rb` | **失敗** |

テスト失敗時のメッセージは下記の通りです。

```
    Failure/Error: is_expected.to eq(Pathname.new("/base_path/backup.tmp"))

       expected: #<Pathname:/base_path/backup.tmp>
            got: #<Pathname:/tmp/backup.tmp>
```

`/base_path` を期待している箇所が `/tmp` に変わっています。

### 原因と修正方法

Ruby では定数の再代入が可能で、警告は出るものの実行は継続されます。この特性により、テストの実行順序によって結果が変わる Flaky Test が発生します。問題となっているファイル `writer_spec.rb` の下記の部分に注目してください。

```ruby
  before do
    Loader::BASE_PATH = "/tmp"
  end
```

これは、ファイルの読み込み順序によっては1回目の `Loader::BASE_PATH` の定義です。しかし、事前に Loader を読み込んでいる場合は2回目の定義となり、1回目の定義を上書きしてしまいます。この改変は、`rspec` コマンドが終了するまで継続します。これが、テスト失敗の原因です。

これを修正するには、以下のように [stub_const](https://rspec.info/features/3-13/rspec-mocks/mutating-constants/) を使って置き換えます。

```ruby
  before do
    stub_const("Loader::BASE_PATH", "/tmp")
  end
```

stub_const は、それを宣言したスコープを出る時に定数の置き換えを元に戻します。これにより、警告を出すことなく定数を置き換え、副作用も解消できます。

## 事例2：クラスの再定義
### サンプルコード

以下の例はクラスの再定義によって起きる Flaky Test の例です。SimpleModule, SpecialModule の二つのモジュールがあり、それぞれテストコードを持っています。

```ruby
# simple_module.rb
module SimpleModule
  def to_s
    "simple!"
  end
end

# special_module.rb
module SpecialModule
  def hello
    "hello: #{self}"
  end
end

# simple_module_spec.rb
RSpec.describe "SimpleModule" do
  before do
    class Dummy
      include SimpleModule
    end
  end

  it do
    expect(Dummy.new.to_s).to eq("simple!")
  end
end

# special_module_spec.rb
RSpec.describe "SpecialModule" do
  before do
    class Dummy
      include SpecialModule
    end
  end

  it do
    expect(Dummy.new.hello).to be_start_with("hello: #<Dummy:")
  end
end
```

この時 `simple_module_spec.rb` と `special_module_spec.rb` はそれぞれ別々に実行すると成功しますが、まとめて実行すると失敗します。

| 実行コマンド                                         | 結果     |
|------------------------------------------------------|----------|
| `rspec simple_module_spec.rb`                        | 成功     |
| `rspec special_module_spec.rb`                       | 成功     |
| `rspec special_module_spec.rb simple_module_spec.rb` | 成功     |
| `rspec simple_module_spec.rb special_module_spec.rb` | **失敗** |

エラーメッセージは下記の通りです。

```
     Failure/Error: expect(Dummy.new.hello).to be_start_with("hello: #<Dummy:")
       expected `"hello: simple!".start_with?("hello: #<Dummy:")` to be truthy, got false
```

`Dummy` クラスの `to_s` は `hello: #<Dummy:` で始まる文字列であることを期待していますが `hello: simple!` を返しています。

### 原因と修正方法

Ruby ではクラスの再定義は、エラーを出しません。オープンクラスとして機能し、メソッドの追加や上書きが発生します。下記のコードに注目してください。

```ruby
  before do
    class Dummy
      include SimpleModule
    end
  end
```

テストコードはクラス `Dummy` を定義しています。これは、スコープを抜けた後も残り続けます。他のテストケースで同じ名前のクラス `Dummy` を使い回していると、クラスオープンを繰り返して拡張することになり、結果として Flaky Test の原因となることがあります。

改善策としては匿名クラスを使います。stub_const と組み合わせると、同じクラス名でも、安全に名前づけできます。

```ruby
RSpec.describe "SimpleModule" do
  before do
    dummy_class = Class.new do
      include SimpleModule
    end

    stub_const("Dummy", dummy_class)
  end
end
```

上記のコードでは、スコープを抜ける前に Dummy クラスは削除され、副作用を与えません。

## 二分探索を使って副作用を与えている犯人を探す

例えば下記のコマンドを実行して100個のファイルテストしたとします。

```
rspec test001.rb test002.rb ... 省略 ... test099.rb test100.rb
```

この時 test100.rb が失敗しました。test100.rb を失敗させる犯人はどのファイルでしょうか。1ファイルずつ実行すると成功することは確認済みとします。素朴なアイデアとして、下記のように100個に分割実行する手があります。

```
rspec test001.rb test100.rb
rspec test002.rb test100.rb
rspec test003.rb test100.rb
... 省略 ...
rspec test098.rb test100.rb
rspec test099.rb test100.rb
```

コマンドのいずれか1つ失敗したならば、副作用を与えている犯人は確定できます。この方法の弱点は、手間がかかることです。また、犯人が3個のファイルの組み合わせだった場合には迷宮入りになってしまいます。

よりスマートな方法としては二分探索を使う方法があります。最初に、下記の2つのコマンドを実行します。

```
rspec test001.rb test002.rb ... 省略 ... test50.rb test100.rb
rspec test051.rb test052.rb ... 省略 ... test99.rb test100.rb
```

成功したコマンドに含まれているファイルは無罪です。失敗したコマンドに含まれているファイルが容疑者となります。これをさらに半分に分割します。同じことを繰り返していけば、わずかな回数で犯人を見つけることができます。二分探索は完璧ではありませんが、複数犯の場合でも、ある程度機能します。

## おわりに

Flaky Test が発生する具体的な事例を紹介し、より現実的なテクニックとして二分探索を使った Flaky Test の調査方法についても紹介しました。しかし、順序以外の原因が潜んでいることもあります。例えば [parallel_test](https://github.com/grosser/parallel_tests) が原因だったというケースもあります。

Flaky Test はいくら調べても再現せず、糸口さえ掴めないこともあります。どうしても解決できない場合は、諦めることも検討しましょう。コメントを残しつつ、スタブを使って安定させる、ペンディングする、といった逃げ道もあります。もしチームでFlaky Testに悩まされている場合は、この記事で紹介した手法を使って、解決に取り組んでみてください。
