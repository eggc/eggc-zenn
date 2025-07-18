---
title: "Warp 入門"
emoji: "🖥️"
type: "tech"
topics:
  - "Warp"
  - "terminal"
  - "AI"
published: true
published_at: "2025-07-17 12:00"
hugo_date: "2025-07-17T12:00:00+09:00"
---

## はじめに

私にとってターミナルといえば、単なる道具の一つでしかありませんでした。カスタマイズして手に馴染ませるのが楽しく、昔は PS1 環境変数や ASCII 制御文字でプロンプトを変えたり、peco で選択インターフェースをカスタマイズしたり、補完プラグインを入れたりしていました。tmux を使っていたこともありました。しかし、環境が変わっていく中でこれらの設定のメンテナンスの手間に疲れて、オールインワンの Warp に乗り換えました。

それからしばらくの間、Warp を使いこなせずにちょっと綺麗なターミナルとして使っていました。しかし、ドキュメントを拾い読みしてみると、もはや「単なる一つの道具」「ちょっと綺麗なターミナル」とはいえないほど、かなり強力な機能がサポートされていることが分かりました。この記事では、次世代のターミナル Warp を紹介します。

## インストール

Windows、Mac 両方をサポートしています。下記サイトからインストーラーをダウンロードできます。

https://www.warp.dev/

アプリケーションを起動すると、ログインを求められます。初回利用時はアカウントを作る必要があります。

## 機能の紹介
### Warp AI

Warp を起動すると、普通のターミナルのように、入力待ち状態となります。ここで ls などのコマンドを叩いても良いのですが、命令文や質問文を与えると自動的に AI への命令と解釈して AI エージェントが動きます。音声入力にも対応しています。以下の命令・質問を試してみましたが、大体期待する通りの結果になりました。

- What is the tar command?
- Find GitHub settings in this repository.
- Copy README.md and rename it to readme-clone.md.

ただし日本語は自動判定できないようです。また英語でも命令として解釈されないものがあります。その場合は command+i を使って明示的に agent mode に切り替えると AI エージェントに入力を与えることができます。

Warp は AI チャットでなく AI エージェントのインターフェースとして働きます。よってファイルの読み書きやその他の操作も可能です。デフォルトで最低限の禁止設定は登録されています。rm sh bash などは基本的に実行前の確認を取る設定となっています。実際に、作ってみたファイルを下記のように削除させてみましょう。

```
Use the rm command to remove the file.
```

確認のメッセージが出て待ち状態となります。このように、入力待ちになっていたり、エラーが出ていたりしたら、タブのアイコンが変化します。タブを隠していたりするとポップアップ通知が出ることもあります。続いて、コードを書かせる実験をしてみましょう。

```
Write a implementation of fizzbuzz in fizzbuzz.rb
```

こうして AI が書き出したファイルにカーソルを当てると Open in Warp というメニューが表示されます。これをクリックすると、組み込みエディタでそのまま編集できます。各プログラミング言語のシンタックスハイライトも有効です。コンテキストを切らさずに操作できるのでとても強力です。

### Warp Drive

Warp では操作をテキスト化してクラウド保存できます。保存したオブジェクトはアカウントに紐づくので環境が違っても再利用できます。アカウントでなくチームに紐づければ複数人での共有ができます。保存できるオブジェクトとしては下記のようなものがあります。

- **workflows**: コマンドのエイリアスです。私はエイリアスは bashrc に書くので使ったことがありません。ただ、workflows はセッションを跨いで利用できるので ssh した後に使うといったユースケースでは役立ちそうです。
- **notebooks**: markdown ドキュメントです。中に含まれているコマンドはターミナルに直接貼り付けて実行できます。環境構築の手順書に使えば効率よくなりそうです。
- **prompt**: AI へ向けた定型文です。定型文の中に変数を使うこともできます。
- **Environment variable**: 環境変数です。複数人でテスト用のAPIキーを共有するといったユースケースで役立ちそうです。1Password などのパスワードマネージャーとの連携もあります。その他、セキュリティを考慮した設定もあるようです。

Warp Drive に保存したオブジェクトにアクセスするには command+p でコマンドパレットを開き、横断的に検索できます。エイリアスやプロンプトを選択すれば、すぐに再利用できます。

私はチームレベルで Warp を活用するところまではできていませんが、うまく利用できればドキュメントを Warp Drive に寄せることで効率的に作業できるかもしれません。

### Modern UX and Text Editing

Warp はシェルに対する一般的なターミナルとしても様々な機能を持っています。例えば補完機能です。かっこを入力したとき、対応するかっこを自動挿入します。また GitHub Copilot のようなサジェスト機能も自動で働きます。提案内容はグレーで表示されます。→ を押すとこの提案を受け入れます。

Warp はコマンド履歴をブロック単位で管理し、操作する機能を持ちます。コマンドのブロックとはコマンド本体と、その結果をひとまとめにしたものです。command+↑↓ でコマンドブロックの履歴を辿れます。下記のショートカットで操作すれば、スクロールしてコピペするよりも、はるかに高速に操作できます。

| キー                | 結果                       |
|---------------------|----------------------------|
| command+c           | コマンドと結果両方をコピー |
| command+shift+c     | コマンドのみコピー         |
| command+alt+shift+c | 結果のみコピー             |

### そのほかのちょっとした機能紹介

- command+d でウィンドウ縦分割、command+shift+d で横分割できます
- コマンドやオプション引数にカーソルを当てると、ドキュメンテーションが表示されます
- コマンド入力中でもシンタックスハイライトされ、エラー箇所は赤線が引かれます
- Git 管理されたディレクトリに移動すると、プロンプトにブランチ情報が表示されます

## まとめ

Warp は従来のターミナルとはいえないほど、AI との協働やチーム開発を支援する統合開発環境に近い存在になっています。チームに停滞感がある場合は Warp Drive の切り口からチーム作業を効率化するのも面白いと思います。

また、AI など流行りの機能を使わないとしても、サジェストやハイライト機能などは無意識に使えます。特にこだわりなくデフォルトのターミナルアプリや、IDE組み込みのターミナルばかり使ってきた、という方は試しに使ってみてください。
