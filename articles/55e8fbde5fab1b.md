---
title: "ruby-lsp による Rubocop 違反の自動修正 in Emacs"
emoji: "💬"
type: "tech"
topics:
  - "ruby"
  - "emacs"
  - "lsp"
published: true
published_at: "2024-08-02 00:39"
hugo_date: "2024-08-02T00:39:00+09:00"
---

## はじめに

Ruby の language server といえば [Solargraph](https://solargraph.org/) でした。しかし、同僚の間では、あまり評判の良いものではありませんでした。私も数年前に少し触ったことがあるのですが、パフォーマンスが悪く、大型のプロジェクトでこれを使うと固まってしまうことが多かったように思います。これに代わる language server として [ruby-lsp](https://github.com/Shopify/ruby-lsp) が開発されました。これまでに多くの場面で Ruby に貢献してきた Shopify が積極的に開発しており期待の高いプロダクトです。Ruby3の型サポートや、 Rubocop とのインテグレーション機能を備えています。長らく使ってみたいと思っていたのですが Emacs のドキュメントがほとんどなく、インターネット上でも Visual Studio Code を使う前提の記事ばかりです。そこで、この記事では Emacs で ruby-lsp を使う方法をまとめ、その便利さを紹介してみたいと思います。

## インストールと設定

はじめにインストールと設定方法を説明します。基本的には language server である ruby-lsp とそのクライアントを Emacs にインストールするだけです。

### ruby-lsp のインストール

基本的には `gem install ruby-lsp` によってインストールするだけです。プロジェクトに関わる全員が必要とするものではないので、Gemfile で管理する必要はないでしょう。インストールできたら `ruby-lsp` が実行できることを確認します。

```
ruby-lsp
Ruby LSP> Skipping custom bundle setup since /Users/eggc/my_project/.ruby-lsp/Gemfile.lock already exists and is up to date
Ruby LSP> Running bundle install for the custom bundle. This may take a while...
Ruby LSP> Command: (bundle check || bundle install) 1>&2
The Gemfile's dependencies are satisfied
```

メッセージからわかるように ruby-lsp はライブラリを追跡するために Gemfile を読み取り、不足するものがあれば bundle install を実行します。エラーがないことを確認したら `Control + c` で停止させましょう。

`rbenv` を利用している場合はプロジェクトごとに ruby-lsp を切り替えれるように設定を作るのが最適ですが、今のところ私は一個のプロジェクトしか扱っていないので、それ専用の設定だけで済ませることにしました。プロジェクトごとの設定が必要になったら改めてまとめてみようと思います。

:::details 補足：Docker を使う場合

元々、私は Docker で開発していたのですが、コードジャンプができなかったため Docker を使うこと自体を諦めることにしました。原因はホストとコンテナのファイルパスの構成が違うためです。コードジャンプは dumb-jump などで補うので必要ない、と言う方は、下記の記事の通りにすれば LSP を使うことができるでしょう。

https://2metz.fr/blog/configuring-emacs-eglot-lsp-with-docker-containers/

コードジャンプを無効にするには下記の設定を使います。ruby 以外の lsp-mode でも無効になってしまうので、 typescript など他の language server を使う方は何らかの工夫をする必要がありそうです。

```lisp
(setq lsp-enable-xref nil)
```

:::

### LSP クライアントのインストールと設定

Emacs での LSP クライアントは eglot と lsp-mode があります。eglot は Emacs 本体に同梱されており、機能もシンプルなので使いやすいのですが、私の環境だと ruby-lsp がうまく動作しなかったため lsp-mode を使用することにしました。

下記のように設定しました。あらかじめ list-package から関連パッケージをインストールしておく必要があります。use-package の ensure 機能を使って自動的にインストールすることもできます。

```lisp
(use-package lsp-mode
  :init
  (setq lsp-keymap-prefix "s-l")　;; Mac のコマンドキー + l を使うことにしました
  (setq lsp-disabled-clients '(rubocop-ls)) ;; ruby-lsp がインストールされていても rubocop が優先されてしまうので、無効にします
  (setq lsp-headerline-breadcrumb-enable nil) ;; パンクズリストは文字化けしてしまったので無効化しました
  :hook
  ((ruby-mode . lsp)
   (lsp-mode . lsp-enable-which-key-integration))
  :commands lsp)

(use-package lsp-ui :commands lsp-ui-mode) ;; LSP の情報が少し見やすくなります
(use-package lsp-ivy :commands lsp-ivy-workspace-symbol) ;; 私は選択UIとして ivy を利用しているのでこれをインストールしました
```

この後、適当な ruby のファイルを開いてモードラインを確認しましょう。下記のように `LSP[ruby-lsp-ls:...]` と表示されたら成功です。

![](https://storage.googleapis.com/zenn-user-upload/4eb65b3c17a4-20240802.png)

うまく動作しない場合は `*ruby-lsp-ls::stderr*` のバッファを確認してみましょう。

## コード診断(Diagnotics)と自動修正(Formatting)

LSP はコード診断と自動修正の機能を提供します。ruby-lsp でもこれらの機能を使ってみましょう。ここでは Rubocop のデフォルトルールに違反した下記のサンプルコードを利用します。

```ruby
class DummyUser
  def name
    string = "hoge"
  end
end
```

違反内容は下記のとおりです。まずは、これらが検出できることを確認しましょう。

1. Style/Documentation: Missing top-level documentation comment for class DummyUser.
2. Style/FrozenStringLiteralComment: Missing frozen string literal comment.
3. Lint/UselessAssignment: Useless assignment to variable - string.
4. Style/StringLiterals: Prefer single-quoted strings when you don't need string interpolation or special symbols.

上記のファイルを開いて LSP を有効にすると、違反箇所は浪線で示されます。カーソルを当てると、詳しい違反の内容を見ることができます。サンプルコードで試してみると、下記のように表示されました。

![](https://storage.googleapis.com/zenn-user-upload/b0a1b1c75f21-20240801.png)

これらの違反箇所の表示には Emacs の flycheck と言う拡張機能が働いています。flycheck には全てのエラーを表示するコマンドがあるので、これを使ってみましょう。`M-x flycheck-list-errors` を実行します。下記のように、全ての違反のリストが表示されました。

![](https://storage.googleapis.com/zenn-user-upload/efcda47252d3-20240801.png)

このリストには Rubocop のチェック以外にも、標準の Ruby の警告も表示されていることがわかります。わざと文法エラーを発生させてみれば、Rubocop 以外の違反を正しく検知できることがわかるでしょう。

![](https://storage.googleapis.com/zenn-user-upload/c346e84a0793-20240801.png)

### lsp-format-buffer による修正

次は、これに自動修正を実行しましょう。 `M-x lsp-format-buffer` を実行します。下記のようになりました。ルール違反(4)の文字列リテラルが修正されたことがわかります。

![](https://storage.googleapis.com/zenn-user-upload/f38e6f3af361-20240801.png)

Rubocop の自動修正はルール(2)(3)(4)に対して有効なはずですが、ここでは(4)のみが適用されました。どうやら ruby-lsp は `rubocop --autocorrect-all` ではなく `rubocop --autocorrect` のみを利用しているようです。

`--autocorrect-all` に関する要望と議論は下記の issue で見ることができます。

https://github.com/Shopify/ruby-lsp/issues/704

### code action による修正

さて、lsp-format-buffer による一括自動修正では(3)を改善できませんでした。ここで LSP の機能 code action を使ってみましょう。問題のファイルを全範囲選択し、`M-x lsp-execute-code-action` を実行します。code action の一覧が表示されました。この中に `Autocorrect Lint/UselessAssignment` が含まれていることがわかります。

![](https://storage.googleapis.com/zenn-user-upload/d51492b8a89f-20240801.png)

選択して実行しましょう。自動的に不要な変数が削除されました。このように、一括修正できない場合は範囲選択して一つずつ自動修正していくことができます。その他の code action としては `Refactor: Extract Variable` や `Refactor: Extract Method` といった機能があります。これらはルール違反の修正ではありませんが、リファクタリングの作業の一部を自動化してくれるでしょう。

## 終わりに

この記事では ruby-lsp のインストール方法と、コード診断、自動修正について説明しました。まだ私も試せてない機能として定義ジャンプ、ドキュメント、コード補完の機能があります。Rails のサポートもあるようです。これらについても便利だと感じた部分があれば後日まとめてみたいと思います。
