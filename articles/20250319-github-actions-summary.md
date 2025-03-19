---
title: "RSpec の結果を Github Actions Summary に出力する"
emoji: "🚀"
type: "tech"
topics:
  - "rspec"
  - "githubactions"
published: true
published_at: "2025-03-19 16:07"
hugo_date: "2025-03-19T16:07:00+09:00"
---

## はじめに

最近、私は CI を実現するために GitHub Actions で RSpec を並列実行するようにしました。しかし、並列化したことで、ノードごとにテストの情報が分散しているため、情報収集がやりづらくなっていると感じました。具体例を見てみましょう。下記のスクリーンショットは RSpec を並列実行して失敗したアクションのサマリーです。

![](https://storage.googleapis.com/zenn-user-upload/a7e841cdc82e-20250319.png)

全てのノードが失敗しているのですが、どれも `Process completed with exit code 1.` というメッセージを出力しているだけです。これはなんら手掛かりになりません。テストが失敗した原因を調べるには、それぞれのノードのログを一つずつ調べなくてはいけません。

今回、私は [GitHub Actions の Summary の機能](https://docs.github.com/ja/actions/writing-workflows/choosing-what-your-workflow-does/workflow-commands-for-github-actions#adding-a-job-summary) を使って、一覧性のある表示に改善しました。全てのノードの RSpec のログを出力し、どのテストが失敗しているのか確認しやすくなりました。

![](https://storage.googleapis.com/zenn-user-upload/af00889cf2b6-20250319.png)

この記事では、具体的な設定方法を紹介します。

## 修正方法

### 修正前の設定ファイル

まず、 RSpec を並列実行する設定ファイルの例を下記に記載します。この例では `matrix` を使って4並列でテストを実行しています。テスト分散については[GitHub Actions でテストを並列化して CI 時間を短縮する | Gunosy テックブログ](https://tech.gunosy.io/entry/actions_parallel) で詳しく紹介されています。ここでは、そのうち最もシンプルな手法を選択しています。

```yaml
# 注：この設定は簡略化しています。プロジェクトに合わせて修正してください。
jobs:
  rspec:
    strategy:
      fail-fast: false
      matrix:
        id: [0, 1, 2, 3]
    runs-on: ubuntu-latest
    services:
      db:
        image: oracle
    steps:
      - name: Check out repository code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
      - name: Install dependencies
        run: bundle install --jobs 4 --retry 3 --path vendor/bundle
      - name: Run RSpec
        run: |
          TESTS=$( \
            find spec -type f -name *_spec.rb | \
              awk -v n=${{ strategy.job-total }}  \
                -v i=${{ matrix.id }} \
                '{ if((NR-1)%n == i){ print $0 } }'
          )
          echo $TESTS

          bundle exec rspec $TESTS
```

### 修正のポイント

GitHub Actions の Summary 機能は、環境変数 `$GITHUB_STEP_SUMMARY` に文字列を書き出すだけで、簡単に使うことができます。テスト実行結果をそのまま出力する場合は、下記のコマンドを実行します。

```sh
echo `bundle exec rspec $TESTS` >> $GITHUB_STEP_SUMMARY
```

これだけで、最低限は動作します。しかし、この方法には二つ欠点があります。一つは、レイアウトが崩れると言う問題です。環境変数 `$GITHUB_STEP_SUMMARY` にセットされた文字列は、全て markdown として書き出されます。RSpec の標準的な出力フォーマットは markdown ではないため、ひどい見た目になってしまいます。

markdown で書き出された時にレイアウト崩れを防ぐにはバッククォート3個で囲むのが良いでしょう。

```sh
# テストを実行し RESULT に結果を格納する
RESULT=`bundle exec rspec $TESTS`

# RESULT をバッククォート3個で囲む
echo "\`\`\`\n$RESULT\n\`\`\`" >> $GITHUB_STEP_SUMMARY
```

ここまでの改善によってサマリー機能が動作するようになりましたが、まだ一つ問題があります。それは「テスト失敗時に、サマリーが出力されない」という問題です。この原因は、コマンドのステータスコードにあります。

rspec コマンドは一つでもテストが失敗するとステータスコード1で終了します。そして GitHub Actions で動作しているシェルはステータスコード1のコマンドが実行された時点でプロセスを終了させます。つまり、上記の設定は rspec に失敗している時、環境変数 `$GITHUB_STEP_SUMMARY` に結果を書き出す前に終了してしまうと言うことです。

この問題を改善するにはシェルのビルトインコマンド `set +e` を使います。このコマンドを実行すると、シェルスクリプトのエラーは無視されるようになります。これによって `rspec` 失敗時もコマンドを継続し、環境変数 `$GITHUB_STEP_SUMMARY` にサマリーを書き込めるようになります。ただし、代わりに、自分でエラーハンドリングをすることを忘れないようにしてください。ここまでで説明した改善をまとめた設定ファイルは以下の通りです。この設定を使うと冒頭で紹介したサマリーを得ることができます。

```yaml
# 注：この設定は簡略化しています。プロジェクトに合わせて修正してください。
jobs:
  rspec:
    strategy:
      fail-fast: false
      matrix:
        id: [0, 1, 2, 3]
    runs-on: ubuntu-latest
    services:
      db:
        image: oracle
    steps:
      - name: Check out repository code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
      - name: Install dependencies
        run: bundle install --jobs 4 --retry 3 --path vendor/bundle
      - name: Run RSpec
        run: |
          TESTS=$( \
            find spec -type f -name *_spec.rb | \
              awk -v n=${{ strategy.job-total }}  \
                -v i=${{ matrix.id }} \
                '{ if((NR-1)%n == i){ print $0 } }'
          )
          echo $TESTS

          # テストを実行するがエラーとなってもここで停止させない
          set +e
          RESULT=`bundle exec rspec $TESTS`
          STATUS=$?
          set -e

          # 標準出力だけでなく Github Actions Summary にも出力
          echo $RESULT
          echo "\`\`\`\n$RESULT\n\`\`\`" >> $GITHUB_STEP_SUMMARY

          # ステータスが正常値でない場合はエラーを発生させる
          if [ $STATUS -ne 0 ]; then
            exit $STATUS
          fi
```

## おわりに

この記事では、GitHub Actions で並列実行している RSpec のテスト結果を Action Summary に出力する方法を紹介しました。Action Summary を活用することで、複数のノードに分散されたテスト結果を一箇所でまとめて確認できるようになり、テスト失敗時の原因調査を効率化できるはずです。いくつか、応用のアイデアを用意しました。お時間のある方は考えてみてください。

- コマンド `rspec` の出力は冗長であるため、不要部分を削る
- コマンド `rspec` の `--format` オプションを使ってカスタムフォーマットで出力し markdown に対応する
- RSpec 以外のテストフレームワーク、例えば JavaScript テストフレームワークにも適用する

皆さんのCIパイプラインの改善に、この記事が少しでも参考になれば幸いです。
