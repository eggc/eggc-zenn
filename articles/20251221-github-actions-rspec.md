---
title: "GitHub Actions 入門 〜RSpec自動化事例〜"
emoji: "🗓️"
type: "tech"
topics:
  - "GitHub Actions"
published: true
published_at: "2025-12-21 00:00"
hugo_date: "2025-12-21T00:00:00+09:00"
publication_name: "litalico"
tags: ["advent-calendar"]
---

## はじめに

この記事は [LITALICO Advent Calendar 2025](https://qiita.com/advent-calendar/2025/litalico) シリーズ1の21日目の記事です。

私が入社してまもなく任された仕事は、Ruby on Rails アプリケーションの自動テスト環境を構築することでした。この記事では、そこで作成した実際の設定例を見ながら細かく説明していきます。まだ GitHub Actions を触ったことがない方でも読める内容にしているつもりなので、ぜひ読んでみてください。

## 設定ファイルの構成要素

GitHub Actions の設定ファイルはYAMLで記述し `.github/workflows/` に配置します。設定ファイルは、下記の要素で構成されています。

- ワークフロー：1個のYAMLファイルに書かれた仕事の計画のことです。
- ジョブ：1台のランナー（サーバー）で行う仕事のことです。
- ステップ：ジョブに含まれる詳細な指示のことです。

![](https://storage.googleapis.com/zenn-user-upload/6e44f0a7f4ba-20251202.jpg)

今回のテーマ「RSpec自動化」を実現する GitHub Actions 設定はワークフロー1個、ジョブ1個、ステップは7個で構成しています。

## ワークフロー設定

まず、自動テストのワークフロー設定を見てみましょう。ここには自動テストの発火タイミングなどの設定があります。

```yaml
name: RSpec
on: [pull_request]
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```


| フィールド名           | 設定値の例                         | 解説                                                                                                                                   |
|:-----------------------|:-----------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------|
| **`name`**             | `RSpec`                            | このワークフローの名前です。GitHub Actions の一覧画面などで表示されます。                                                              |
| **`on`**               | `[pull_request]`                   | このワークフローが動き出すトリガーを設定します。今回はプルリクエスト作成・更新時です。                                                 |
| **`concurrency`**      | -                                  | 連続実行時の制御を行う設定ブロックです。無駄な実行の抑制に使います。                                                                   |
| └ `group`              | `${{ ...workflow }}-${{ ...ref }}` | 同じワークフローかどうかを示すIDです。今回は同じブランチであれば同じとします。                                                         |
| └ `cancel-in-progress` | `true`                             | 上記の ID と一致するようなトリガーが発火した場合は古いワークフローをキャンセルして新しい方だけ実行します。コスト節約のための設定です。 |

## ジョブ設定

次に、自動テストのワークフローに含まれるジョブ設定を確認します。今回はアプリケーションコンテナ、データベースコンテナの2つで構成します。

![](https://storage.googleapis.com/zenn-user-upload/47a3872e7801-20251202.jpg)

```yaml
jobs:
  rspec:
    strategy:
      fail-fast: false
      matrix:
        id: [0, 1, 2, 3]
    runs-on: ubuntu-latest
    container:
      image: my-webapp-image:latest
      env:
        ORACLE_DATABASE_TEST: //oracle:1521/FREEPDB1
        ORACLE_USERNAME_TEST: test_user
        ORACLE_PASSWORD_TEST: test_password
        RAILS_ENV: test
        NLS_LANG: JAPANESE_JAPAN.UTF8
        TZ: Asia/Tokyo
    services:
      oracle:
        image: my-oracle-image:latest
        ports:
          - "1521:1521"
        env:
          NLS_LANG: JAPANESE_JAPAN.UTF8
          ORACLE_PWD: test_password
          TZ: Asia/Tokyo
```

今回はテスト実行時間の短縮（高速化）のため、`matrix` 機能を使って4台のホストランナーを同時に動かします。ランナーごとに担当するテストファイルを分担させるために、後で紹介するステップの設定では `matrix.id` を使って処理を分岐させています。

| フィールド名    | 設定値の例         | 解説                                                                                                                   |
|:----------------|:-------------------|:-----------------------------------------------------------------------------------------------------------------------|
| **`jobs`**      | -                  | ワークフローで実行するジョブの一覧を定義します。今回は rspec というジョブ1個だけです。                                 |
| **`strategy`**  | -                  | ジョブの実行戦略です。今回はテストを並列実行（Matrix実行）するために使用しています。                                   |
| └ `fail-fast`   | `false`            | 並列実行中に1つが失敗しても処理を継続します。これを書かないと、1個失敗した時点で全て失敗となり強制終了してしまいます。 |
| └ `matrix`      | `id: [0, 1, 2, 3]` | 同じジョブを4つのノード（並列数4）で実行するための変数設定です。並列数を増減したい場合はこの数を修正してください。     |
| **`runs-on`**   | `ubuntu-latest`    | ジョブを実行するランナー（サーバー）を指定します。                                                                     |
| **`container`** | -                  | ランナーで動作するメインのコンテナを指定します。今回は Rails アプリケーションのコンテナです。                          |
| └ `env`         | -                  | コンテナ内で有効になる環境変数を列挙します。                                                                           |
| **`services`**  | -                  | ランナーで動作するサブのコンテナを指定します。今回の例では Oracle データベースのみです。                               |

### ステップ設定

最後に、ジョブの中で実行される「ステップ」を見ていきましょう。全てのステップはメインコンテナの中で順次実行されます。ステップは主に `run` か `uses` のいずれかで構成されています。

- **run**: 具体的なシェルコマンド（`mkdir`, `bundle exec rspec` など）を実行する場合に使います。
- **uses**: アクション（誰かが作った汎用的な機能）を実行する場合に使います。[GitHub Marketplace](https://github.com/marketplace) で公開されているものを利用できます。


```yaml
steps:
  - name: Check out repository code
    uses: actions/checkout@v4
    with:
      ref: ${{ github.head_ref }}
  - name: Create directory
    run: mkdir -p vendor/bundle
  - name: Restore cache files
    uses: actions/cache@v4
    with:
      path: vendor/bundle
      key: ${{ runner.os }}-${{ runner.arch }}-gems-${{ hashFiles('**/Gemfile.lock') }}
      restore-keys: |
        ${{ runner.os }}-${{ runner.arch }}-gems-
  - name: Install dependencies
    run: bundle install --jobs 4 --retry 3 --path vendor/bundle
  - name: Wait for import tables
    run: bin/ci-db-check
  - name: Run Migrations
    run: bundle exec rails db:migrate
  - name: Run RSpec
    run: bin/ci-rspec
    env:
      JOB_TOTAL: ${{ strategy.job-total }}
      JOB_ID: ${{ matrix.id }}
```


| フィールド名 | 設定値の例                    | 解説                                                           |
|:-------------|:------------------------------|:---------------------------------------------------------------|
| **`name`**   | `Check out repository code`   | ステップのタイトルです。実行ログで見やすくするために付けます。 |
| **`uses`**   | `actions/checkout@v4`         | 「アクション」を呼び出して使います。                           |
| **`with`**   | `ref: ${{ github.head_ref }}` | `uses` で使う機能に渡すオプション設定（引数）です。            |
| **`run`**    | `bundle install ...`          | シェルコマンドを指示します。                                   |
| **`env`**    | `JOB_ID: ${{ matrix.id }}`    | このステップ内だけで有効な環境変数を定義します。               |

次は、各ステップを簡単に説明していきます。

#### Check out repository code

メインコンテナはソースコードなどを持たない状態で立ち上がります。テストをするためにはソースコードが必要です。そこでGitHub上のリポジトリからソースコードを取得します。

ここでは `actions/checkout@v4` という公式アクションを使っています。設定 `ref: ${{ github.head_ref }}` により対象のブランチをプルリクエスト作成ブランチとしています。

https://github.com/actions/checkout

#### Create directory

依存ライブラリ(Gem)をインストールするためのディレクトリを作成します。

#### Restore cache files

以前のビルドで保存された Gem のキャッシュを復元し、`bundle install` の時間を短縮します。

GitHub Actions のキャッシュには「キャッシュスコープ」という概念があるため注意が必要です。「デフォルトブランチで作ったキャッシュは他のブランチでも利用できるが、その逆はできない」という仕様になっています。そのため、私たちはデフォルトブランチでキャッシュを作るだけのワークフローを別途作成して対策しています。詳しい解説は以下の記事が参考になります。

[GitHub Actions は、 default ブランチで実行しておかないとキャッシュが効かないことがある](https://zenn.dev/mixi/articles/build-github-action-workflow-on-default-branch)

#### Install dependencies

`Gemfile` にしたがって、必要な Ruby ライブラリ（Gem）をインストールします。キャッシュが見つかり、差分がない場合はほぼ0秒で終了します。もしアーキテクチャ変更などでキャッシュミスした場合は、全ての Gem をインストールするため数分余計に時間がかかります。

#### Wait for import tables

データベースの初期化（テーブル定義のインポートなど）が完了するまでメインコンテナを待機させます。使用しているDBのカスタムイメージは、起動後にバックグラウンドで初期化処理が走る仕様のため、これが必要です。

具体的なスクリプト `bin/ci-db-check` の中身は割愛しますが、数秒ごとにデータベースの状態をポーリングし、準備完了になったら次のステップへ進むように制御しています。

#### Run Migrations

マイグレーションを実行してデータベースを最新の状態にします。DBイメージに含まれるスナップショット以降に追加されたテーブル変更などを、ここで反映させます。

#### Run RSpec

いよいよ RSpec による自動テストを実行します。ここでは環境変数 `JOB_TOTAL`（総ジョブ数: 4）と `JOB_ID`（現在のジョブ番号: 0~3）をテスト実行スクリプトに渡し、ランナーごとに実行すべきテストファイルを分担させています。実際のテストスクリプトは下記のとおりです。この程度のスクリプトであれば、直接 YAML に記載しても良いのですが、ローカルでの試し打ちや、今後のカスタマイズのために分離してあります。

```
#!/bin/bash

TESTS=$( \
  find spec -type f -name *_spec.rb | \
    awk -v n=$JOB_TOTAL  \
      -v i=$JOB_ID \
      '{ if((NR-1)%n == i){ print $0 } }'
)

bundle exec rspec $TESTS
```

テストファイルの分配戦略には様々なものがあります。より高度な並列化に興味のある方は、下記の記事が参考になると思います。

[GitHub Actions でテストを並列化して CI 時間を短縮する - Gunosy Tech Blog](https://tech.gunosy.io/entry/actions_parallel)

## まとめ

ここまでの内容をまとめたファイルは下記のとおりです。

```yaml
name: RSpec
on: [pull_request]
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  rspec:
    strategy:
      fail-fast: false
      matrix:
        id: [0, 1, 2, 3]
    runs-on: ubuntu-latest
    container:
      image: my-webapp-image:latest
      env:
        ORACLE_DATABASE_TEST: //oracle:1521/FREEPDB1
        ORACLE_USERNAME_TEST: test_user
        ORACLE_PASSWORD_TEST: test_password
        RAILS_ENV: test
        NLS_LANG: JAPANESE_JAPAN.UTF8
        TZ: Asia/Tokyo
    services:
      oracle:
        image: my-oracle-image:latest
        ports:
          - "1521:1521"
        env:
          NLS_LANG: JAPANESE_JAPAN.UTF8
          ORACLE_PWD: test_password
          TZ: Asia/Tokyo
    steps:
      - name: Check out repository code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
      - name: Create directory
        run: mkdir -p vendor/bundle
      - name: Restore cache files
        uses: actions/cache@v4
        with:
          path: vendor/bundle
          key: ${{ runner.os }}-${{ runner.arch }}-gems-${{ hashFiles('**/Gemfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-${{ runner.arch }}-gems-
      - name: Install dependencies
        run: bundle install --jobs 4 --retry 3 --path vendor/bundle
      - name: Wait for import tables
        run: bin/ci-db-check
      - name: Run Migrations
        run: bundle exec rails db:migrate
      - name: Run RSpec
        run: bin/ci-rspec
        env:
          JOB_TOTAL: ${{ strategy.job-total }}
          JOB_ID: ${{ matrix.id }}
```

なお、このコードはサンプルです。実際に動作させるには、ご自身の環境に合わせて以下の調整が必要です。

- コンテナ設定: `my-webapp-image` 等のカスタムイメージを利用する場合、レジストリへのログイン設定や権限確認が必要です。
- 機密情報の管理: データベースのパスワードなどは直接記述せず、GitHub Secrets などを利用するのが好ましいでしょう。
- スクリプトの準備: `bin/ci-rspec` や `bin/ci-db-check` などの独自スクリプトファイルを作成・配置する必要があります。
- その他のコンテナの追加: アプリケーションによっては `services:` に Redis コンテナなど他の連携コンテナを追加してください。

## おわりに

テスト自動化の環境構築は開発における定番のタスクですが、一度作ってしまうと変更頻度が低いため、どうしても属人化しやすい領域です。しかし、特定の誰かだけが触れるのではなく、チームの誰もがこうした改善に取り組める状態を目指したいと思い、今回の記事を執筆しました。この記事が、これから GitHub Actions で自動テスト環境を構築しようとしている方や、既存の設定をメンテナンスしようとしている方の参考になれば幸いです。

最後に少しだけ宣伝です。LITALICOでは現在エンジニアを募集しています。よかったら[エンジニア求人一覧](https://herp.careers/v1/litalico/requisition-groups/ea1f6b9d-88b7-4254-a06a-8cfdd83bd575)を覗いてみてください。
