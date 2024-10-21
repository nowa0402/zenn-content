---
title: "Ryeとキャッシュ機能で快適なCI環境を整備してみた"
emoji: "🐰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "rye", "github", "githubactions"]
published: true
publication_name: "spectee"
---

## 本記事の結論

GitHubActionsでryeのキャッシュを活用したければ`setup-rye`がオススメ
https://github.com/eifinger/setup-rye

## 導入経緯

私の所属するチームでは、コードの品質を保つためGitHubActionsを使用して静的解析や単体テストを実施しています。
現状は特に問題が発生してませんが、今後のプロジェクト拡大を意識し、キャッシュ機能を用いて今のうちから速度改善が見込めないか検証してみました。

## 前提条件

今回のCIはGitHub Actionsで行うものとし、コードをpushしたタイミングで動作させます。
検証内容は、以下の作業がすべて終了した時間を改善前と後で比較します。

- ruff: フォーマット・静的解析
- mypy: 型チェック
- pytest: 単体テスト
- cpell: スペルチェック

検証に使用したリポジトリは、複数の独立したAWS Lambdaサービスが存在しています。
既存CIでは各サービスを並列で実行できるようにしています。

## 改善前の構成

改善前のCIのフローになります。サービス名は仮でA・B・C・Dとしています。

![flow-1](https://storage.googleapis.com/zenn-user-upload/967c514dfa4c-20241015.png)

この状態で実行したところ、結果は1分51秒となりました。

![actions-1](https://storage.googleapis.com/zenn-user-upload/06e364000bed-20241015.png)

### ボトルネック要因はパッケージインストール

このフローの中で一番時間がかかっている要因は`pip`コマンドによるパッケージのインストールです。
これがないと単体テストやmypyによる型チェックが上手く出来ないので必須となります。

例として、あるサービスの単体テストの実行時間をお見せします。
パッケージのインストールは`Install dependencies`のステップで実行しています。

22秒かかっているのが分かります。

![actions-2](https://storage.googleapis.com/zenn-user-upload/07d62e3df2f7-20241015.png)

今のフローは静的解析と単体テストが子ワークフローとなっており、親から呼び出して動いています。
そのため、それぞれのワークフローでパッケージをインストールする必要があり、全体の実行時間に大きく影響しています。

また、今のフローだとパッケージに変更がない場合も毎回インストール作業が必須となります。
今はそれほど大きな時間がかかっているわけではありませんが、プロジェクトが大きくなった場合、実行時間の遅れがプロジェクト全体の進捗に影響する可能性があります。

これらの課題をキャッシュを用いて解決できるか検証していきます。

## Ryeでキャッシュ機能を実装する

GitHub Actionsでパッケージ関係やビルドを早くするならキャッシュがよく使われます。
https://github.com/actions/cache

私たちはRyeでパッケージ管理をしているため、サードパーティ製の`setup-rye`を使うことにしました。

https://github.com/eifinger/setup-rye

サードパーティ製にはなりますが
👇️理由のため比較的安心して使えると思います。

- 本家の[ryeのワークフロー](https://github.com/astral-sh/rye/tree/main/.github/workflows)にも採用されている
- メンテが定期的に行われている（2024年9月末時点）

また、オプションでキャッシュ機能を指定可能で、今回の検証要件を簡単に実現することができます。

### フロー

キャッシュを導入するに伴い、まずは既存のフローから修正を行いました。

修正点は以下のとおりです。

- 親のワークフローで先に仮想環境作成＆パッケージをインストール
- 子のワークフローでキャッシュされた仮想環境を利用
- 単体テストと静的解析を1つのワークフローに統合

これらを反映することで
課題の一つだった「別々のワークフローでパッケージをインストールしている」問題を解決します。

上記を反映させたものが下図のフローになります。

![flow-2](https://storage.googleapis.com/zenn-user-upload/7c1cf516b5f8-20241015.png)

### 実装

それでは実装してみます。
まずはプッシュトリガーで動作する親のワークフローからです。

ポイントは`cache-prefix`でキャッシュのプレフィックスを指定することです。
親で指定したキーを子のワークフローでも指定することで、親で設定したキャッシュの引き継ぎが可能になります。

```yaml:parent.yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: setup rye
        id: setup-rye
        uses: eifinger/setup-rye@v4
        with:
          enable-cache: true
          cache-prefix: rye-cache # ポイント：cacheのprefixを設定する

    # 【参考】キャッシュが取得できたか確認するためのコード
      - name: Do something if the cache was restored
        run: |
          if [[ ${{ steps.setup-rye.outputs.cache-hit }} == "true" ]]; then
            echo "Cache was restored"
          else
            echo "Cache was not restored"
          fi

    # パッケージをインストールし、.venvを作成する
      - name: install dependencies
        run: rye sync
    
    # ここでjobが終了。キャッシュ情報として.venvがGitHubのサーバに保存される

  ci:
    needs: setup
    uses:
      ./.github/workflows/ci.yml # 子ワークフローの静的解析・単体テストを実行する
```

続いて子のワークフローを実装します。Ryeのセットアップは親と一緒です。
ポイントは **「`cache-prefix`を親と一緒にする」** ことです。
これにより、親が先行して作成・キャッシュしてくれたPythonの仮想環境を使うことができます。

```yaml:ci.yaml
name: CI

on:
  workflow_call:

jobs:
  ci:
    strategy:
      matrix:
        ci_target_dir:
          - ./services/A
          - ./services/B
          - ./services/C
          - ./services/D

    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: setup rye
        id: setup-rye
        uses: eifinger/setup-rye@v4
        with:
          enable-cache: true
          cache-prefix: rye-cache # 親と同じプレフィックスを設定する

      - name: install dependencies
        run: rye sync

      - name: run ci for python project
        run: |
          CI_TARGET_DIR=${{ matrix.ci_target_dir }} bash -c scripts/ci.sh

      - name: spell check with cspell
        uses: streetsidesoftware/cspell-action@v6
        with:
          root: ${{ matrix.ci_target_dir }}
          config: ./cspell.json
```

参考までに`ci.sh`とサービスごとの`pyproject.toml`の中身です。

まずはci.sh。サービスディレクトリに移動してコマンドを発行します。

```bash:ci.sh
echo "CI TARGET DIR: ${CI_TARGET_DIR}"
cd ${CI_TARGET_DIR} && rye run ci
```

続いて`pyproject.toml`
`rye run ci`スクリプトはチェイン機能でつなげています。

```toml:pyproject.toml
[tool.rye.scripts]
test = "pytest -vv tests"
check = { chain = ["check:ruff", "check:mypy"] }
"check:ruff" = "ruff check --config pyproject.toml src"
"check:mypy" = "mypy src --config-file=../../pyproject.toml"
ci = { chain = ["check", "test"] }

[tool.ruff]
# ruffのルールは親に配置しているので参照するための設定
extend = "../../pyproject.toml"
```

### 測定

それでは実装したものを測定します。改善前は1分51秒でした。

![actions-3](https://storage.googleapis.com/zenn-user-upload/63ef8e764035-20241015.png)

改善後は1分。**改善前と比べて約50秒改善**となりました！
初回は親側でのインストールがあるため時間がかかっています。

![actions-4](https://storage.googleapis.com/zenn-user-upload/923b65553be6-20241015.png)

2回目以降パッケージの変更がなければ、キャッシュが機能し更に早くなります。

![actions-5](https://storage.googleapis.com/zenn-user-upload/ed763b37de67-20241015.png)

2回目の実行。**更に10秒ほど早くなりました。** 親側でもキャッシュが効いたためです。
詳細画面でも確認してみます。

![actions-6](https://storage.googleapis.com/zenn-user-upload/b7ac05687d67-20241015.png)

仮想環境が存在する旨のメッセージと共にインストールがスキップされているのが分かります。
これで課題だった、パッケージに変更がない場合も毎回インストールが発生する問題が解決されました！

## 注意点

便利なキャッシュ機能ですが、以下のルールがあります。

- 一つのリポジトリで保有できるキャッシュ量に制限があります。
- キャッシュを検索するときに同一キーが無いか検索（なければprefix検索）する

そのため、運用がうまく出来ていないと無駄にキャッシュを生成して全く使われない…みたいなことも起きます。

👇️記事は、他チームの方が共有していただいた記事です。

https://ymmt.hatenablog.com/entry/2024/10/02/222243

つまり、キャッシュ利用時はブランチ運用の考慮も必要です（奥が深い…）。
私の所属するチームはトランクベースをちょっといじったような運用なのでそこまで記事の影響を受けませんが、キャッシュを検討する際は一度記事に目を通しておくと良さそうです。

https://www.docswell.com/s/uta8a/KYDW9P-2024-08-22-github-actions-tips#p13

## まとめ

今回はPythonプロジェクトでRyeとキャッシュを使った快適なCI環境整備を紹介しました。
改善は約1分ほどでしたが、プロジェクトが肥大になっていけばより恩恵を受けられそうな印象を受けました。

また、今回この記事を作成するにあたり、今まで曖昧だったキャッシュ機能の動作概要についてチームメンバー全員で理解を深めることができたと思います。
今後もキャッシュ機能を使って大幅な改善が見込めそうなプロジェクトがあれば活用したいと思います！

## 参考

### キャッシュについて

- [actions/cache](https://github.com/actions/cache)
- [setup-rye](https://github.com/eifinger/setup-rye)

### 運用

- [やんないほうがいいかも、GitHub Actions の setup-xxx での依存キャッシュ保存](https://ymmt.hatenablog.com/entry/2024/10/02/222243)
- [GitHub Actions ステップアップ Tips](https://www.docswell.com/s/uta8a/KYDW9P-2024-08-22-github-actions-tips#p13)
