---
title: "Emacs"
---

## todo: 改善のアイデア

Ruby のコードジャンプは dumb-jump で検索ジャンプしてるけど、同じファイルの中だとわかってる時は dumb-jump 使わずに imenu で探した方が早い。なので M-. は imenu で探した後見つからなかったら dumb-jump を使うみたいな、フォールバックで構成することはできないかなと思っている。

## pr-preview

Emacs から GitHub のプルリクエストにコメントを書き込むことができるらしい。

https://github.com/blahgeek/emacs-pr-review

このパッケージを知ったきっかけ： https://magnus.therning.org/2025-01-19-reviewing-github-prs-in-emacs.html

## gptel

Emacs から各種 AI を呼び出してバッファ内に書き込むことができるらしい。
私は今のところどの API キーを持ってないので使うことはできないけど、
OpenAI のライセンスとか購入して使ってみるのも悪くないと思っている。
しばらくは会社支給の Github Copilot Chat で足りると思う。
ただ、個人のアカウントでは使えないので、そこをどうしようか。

https://github.com/karthink/gptel

## web-mode

erb を編集するときに web-mode にしているが、ペーストしただけでファイル全体のオートインデントがかかってしまうのでこれをやめにしたい。方法は下記の記事の通りにすれば良さそう。

https://blog.shibayu36.org/entry/2016/03/17/183209

## consult-gh

Emacs から GitHub を操作するパッケージとして forge は操作が重くて厳しい感じだったけど内部的に gh コマンドを使った実装（補完機能中心？）が登場していた。まだ日本語情報が少ないみたいなので、使ってみてレポートしたら面白いかもしれない。

https://github.com/armindarvish/consult-gh

## txl

Emacs から Deepl 翻訳を使うパッケージを入れてみた。今のところ、一方向翻訳しかできなかったので逆方向の翻訳もできるようなカスタマイズをしてみたい。

https://github.com/tmalsburg/txl.el

カスタマイズするなら、もっと低水準な機能を持ってる下記パッケージの方がいいかもしれない。ちょっと使いにくさはあるけど・・・。

https://github.com/emacs-openai/deepl

## robe

Ruby のコードジャンプを REPL プロセスを利用して実行するパッケージがある。

https://github.com/dgutov/robe

しかしこれは LSP が流行っている時代的にはマッチしてない設計なので、今後も続いていくかは難しいところ。ruby-lsp の方はかなりアクティブなので、そっちが完成したら robe は必要ないかもしれない。

https://github.com/Shopify/ruby-lsp/graphs/commit-activity

## org-copy-visilbe コマンドの利用

magit-log でファイルパスが長い時に下記のような感じで省略される

```
.../api/version1/hogehoge/fugafuga/timestampscontrollerspec.rb         | 2 +-
```

たまに困るのでフルパスにしてほしい。その場合は shift + 1 でグループを fold したら良い。
`org-copy-visible` というコマンドがある。
これは fold して折り畳んだ文字列をそのまま見た目通りにコピーしたりするのに使える。
org に限らず magit diff でも使える。
