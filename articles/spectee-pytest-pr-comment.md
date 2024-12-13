---
title: "pytestカバレッジをプルリクエストのコメントに表示する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pytest", "github", "githubactions", "rye"]
published: true
publication_name: "spectee"
---

## 導入経緯

[Spectee](https://spectee.co.jp/)でエンジニアをしている和山です。

プルリクエストのレビューにおいて、単体テストコードの妥当性を確認することは重要な観点の一つです。その際、カバレッジの確認が目安として活用されることがあります。
しかし、現状ではカバレッジを確認するために、ローカル環境で実行して出力を確認するか、GitHub Actionsの履歴を参照する必要があり、レビューに必要な手順や時間が増加するという課題が発生していました。

この問題を解決するために、 **プルリクエストのコメントに pytest のカバレッジ結果を自動的に表示する仕組みを導入しました。** この仕組みにより、開発プロセスの透明性と効率性が向上し、レビューの負担を軽減することが期待されます。

## 本記事の結論

サードパーティ製アクションの`Pytest Coverage Comment` を使用する。
カバレッジの出力ファイルをアクションに渡してコメントを行う。
https://github.com/marketplace/actions/pytest-coverage-comment

:::details 公式サンプルはこちら

```yml
name: pytest-coverage-comment
on:
  pull_request:
    branches:
      - '*'

permissions:
  contents: write
  checks: write
  pull-requests: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

        # 1.Pythonセットアップ
      - name: Set up Python 3.8
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest pytest-cov
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

        # 2.pytest実行
      - name: Build coverage file
        run: |
          pytest --junitxml=pytest.xml --cov-report=term-missing:skip-covered --cov=app tests/ | tee pytest-coverage.txt

        # 3.アクション起動
      - name: Pytest coverage comment
        uses: MishaKav/pytest-coverage-comment@main
        with:
          pytest-coverage-path: ./pytest-coverage.txt
          junitxml-path: ./pytest.xml
```

:::

:::details ryeと組み合わせたワークフローはこちら
ディレクトリ構成

```bash
.
└── .github
│   └── workflows
│       ├── ci.yml
│       └── cicd.yml
├── pyproject.toml
├── scripts
│   └── test_cov.sh
├── src              
└── tests  
```

pyproject.toml

```toml:pyproject.toml
[tool.rye.scripts]
ci = { chain = ["check", "test:cov"] }
check = { chain = ["check:ruff", "check:mypy"] }
"test:cov" = "./scripts/test_cov.sh"
"check:ruff" = "ruff check --config pyproject.toml src"
"check:mypy" = "mypy src --config-file=pyproject.toml"
```

scripts/test_cov.sh

```bash:scripts/test_cov.sh
#!/bin/bash
pytest tests/ --cov=src --cov-branch --junitxml=pytest.xml --cov-report=term-missing:skip-covered | tee pytest-coverage.txt
```

.github/workflows/cicd.yml

```yml:.github/workflows/cicd.yml
name: CI/CD Pipeline

on:
  push:
  pull_request:          # 重要:ワークフローの起動条件にプルリクエストを追加

permissions:
  pull-requests: write   # 重要:権限設定
  contents: read        

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
          cache-prefix: rye-cache

      - name: install dependencies
        run: rye sync

  ci:
    needs: setup
    uses: ./.github/workflows/ci.yml  # ci.yml呼び出し

  # 以下ジョブは割愛
```

.github/workflows/ci.yml

```yml:.github/workflows/ci.yml
name: CI

on:
  workflow_call:

permissions:
  pull-requests: write # 重要:権限設定
  contents: read

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: setup rye
        id: setup-rye
        uses: eifinger/setup-rye@v4
        with:
          enable-cache: true
          cache-prefix: rye-cache

      - name: install dependencies
        run: rye sync

      - name: run ci for python project # 重要:CIの実行（静的解析〜テスト）
        run: |
          rye run ci

      - name: comment coverage on pull request
        if: github.event_name == 'pull_request' # 重要:プルリクエストの時にテストカバレッジをコメントする
        uses: MishaKav/pytest-coverage-comment@main
        with:
          pytest-coverage-path: ./pytest-coverage.txt
          junitxml-path: ./pytest.xml
          title: Coverage Report（ServiceA）
          unique-id-for-comment: ServiceA # 同じコメントにテスト結果を追記するための一意なID
          create-new-comment: false # 既存のコメントを更新します。同じコメントにテスト結果を追記するため、新しいコメントは作成しません。
```

:::

## 導入手順 パターン1（公式サンプル）

以下[公式](https://github.com/marketplace/actions/pytest-coverage-comment#example-usage)のワークフローのサンプルです。

```yml
name: pytest-coverage-comment
on:
  pull_request:
    branches:
      - '*'

permissions:
  contents: write
  checks: write
  pull-requests: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

        # 1.Pythonセットアップ
      - name: Set up Python 3.8
        uses: actions/setup-python@v2
        with:
          python-version: 3.8

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest pytest-cov
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

        # 2.pytest実行
      - name: Build coverage file
        run: |
          pytest --junitxml=pytest.xml --cov-report=term-missing:skip-covered --cov=app tests/ | tee pytest-coverage.txt

        # 3.アクション起動
      - name: Pytest coverage comment
        uses: MishaKav/pytest-coverage-comment@main
        with:
          pytest-coverage-path: ./pytest-coverage.txt
          junitxml-path: ./pytest.xml
```

出力イメージは[公式](https://github.com/marketplace/actions/pytest-coverage-comment#output-example)参照
ワークフローの流れは以下の通り

1. Pythonのセットアップ
2. pytestコマンド実行。カバレッジファイル出力
3. アクションの引数にカバレッジファイルを指定する

重要なのはpytestの実行とアクションの実行です。

### pytestコマンドでカバレッジを出力

`pytest`コマンドでテストの実行とカバレッジの出力を行います。

```bash
pytest --junitxml=pytest.xml --cov-report=term-missing:skip-covered --cov=app tests/ | tee pytest-coverage.txt
```

各オプションの説明

- `--junitxml=pytest.xml`: テスト結果をJUnit形式のXMLファイルとして保存
- `--cov-report=term-missing:skip-covered`: カバレッジレポートの形式と内容を指定
  - `term-missing`: ターミナルにカバレッジレポートを出力
  - `skip-covered`: 100%カバレッジのファイルや行をスキップし、未カバー部分だけ表示
- `--cov=app`: コードカバレッジ計測対象ディレクトリ。例では`app`ディレクトリが対象
- `| tee pytest-coverage.txt`: コマンドの出力を標準出力とファイルの両方に記録

実行すると2つのファイルが作成されます。

- pytest.xml
- pytest-coverage.txt

### pytest-coverage-commentアクションを起動

アクションを起動します

```yml
- name: Pytest coverage comment
uses: MishaKav/pytest-coverage-comment@main
with:
    pytest-coverage-path: ./pytest-coverage.txt
    junitxml-path: ./pytest.xml
```

- `pytest-coverage-path`: pytest-coverage.txt出力パス。カバレッジのパーセンテージを判断するために使用
- `junitxml-path`: junitxmlの出力パス。テストの数（成功、失敗など）を判断するために使用

## 導入手順 パターン2（ryeのプロジェクトに組み込む）

先ほどの手順は公式サンプルを利用したものです。
私達のチームは[rye](https://rye.astral.sh/)でプロジェクト管理しています。
公式サンプルを元にryeが使いやすい形に修正しました。特殊なケースかもしれませんが紹介します。

ディレクトリ構成は下記の通り。

```bash
.
├── pyproject.toml
├── scripts
│   └── test_cov.sh  # カバレッジファイル出力用シェルスクリプト
├── src              # プロダクションコードが格納されているディレクトリ
└── tests            # テストコードが格納されているディレクトリ
```

### 手順1.pytestコマンド実行スクリプトの作成と各種設定

前提としてryeでコマンド実行する場合はryeのスクリプト機能を使うと便利です。
pyproject.tomlに設定したスクリプトを`rye run <script>`で動かすことができます。

`test:cov`で動作するようにpyproject.tomlを設定します

```toml:pyproject.toml
[tool.rye.scripts]
"test:cov" = "pytest tests/ --cov=src --cov-branch --junitxml=pytest.xml --cov-report=term-missing:skip-covered | tee pytest-coverage.txt"
```

`rye run test:cov`を実行します

```bash
rye run test:cov
=================================================== test session starts ====================================================
platform darwin -- Python 3.12.2, pytest-8.3.2, pluggy-1.5.0
rootdir: <テスト対象ディレクトリ>
plugins: cov-5.0.0, anyio-4.4.0
collected 0 items                                                                                                          

---------- generated xml file: <プロジェクトルート>/pytest.xml -----------
================================================== no tests ran in 0.01s ===================================================
ERROR: file or directory not found: |
```

パイプ`|`がうまく渡せないようです。
シェルスクリプトファイル経由で動かすように修正します

```bash:script/test_cov.sh
#!/bin/bash
pytest tests/ --cov=src --cov-branch --junitxml=pytest.xml --cov-report=term-missing:skip-covered | tee pytest-coverage.txt
```

スクリプトファイルに実行権限を付与します。

```bash
chmod 755 scripts/test_cov.sh
```

pyproject.tomlファイルの修正を行います

```toml:pyproject.toml
[tool.rye.scripts]
"test:cov" = "./scripts/test_cov.sh"
```

これで実行できるようになりました。

```bash
rye run test:cov
============================= test session starts ==============================
platform darwin -- Python 3.12.2, pytest-8.3.2, pluggy-1.5.0
rootdir: <テスト対象ディレクトリ>
plugins: cov-5.0.0, anyio-4.4.0
collected 41 items

# 以下テスト結果が表示される
```

この後の手順でワークフローを実装します。
CI上でRuffやmypyによる静的解析も行いたい場合はチェインを使うと良いです

```toml:pyproject.toml
[tool.rye.scripts]
ci = { chain = ["check", "test:cov"] }
check = { chain = ["check:ruff", "check:mypy"] }
"test:cov" = "./scripts/test_cov.sh"
"check:ruff" = "ruff check --config pyproject.toml src"
"check:mypy" = "mypy src --config-file=pyproject.toml"
```

ローカル環境で`rye run ci`で静的解析とテストが行われます

pytest.xmlとpytest-coverage.txtファイルが出力されれば成功です。
この時点でのディレクトリ構成は以下の通り

```bash
.
├── pytest.xml
├── pytest-coverage.txt
├── pyproject.toml
├── scripts
│   └── test_cov.sh  # 追加
├── src              
└── tests  
```

pytest.xmlとpytest-coverage.txtは.gitignoreに追記し、コミット対象外にすると良いです。

### 手順2. ワークフローファイル整備

本例はcicd.ymlファイルがci.ymlを呼び出す形にします。

```bash
.
└── .github
   └── workflows
       ├── ci.yml
       └── cicd.yml
```

まずはcicd.ymlの設定

```yml:cicd.yml
name: CI/CD Pipeline

on:
  push:
  pull_request:          # 重要:ワークフローの起動条件にプルリクエストを追加

permissions:
  pull-requests: write   # 重要:権限設定
  contents: read

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
          cache-prefix: rye-cache

      - name: install dependencies
        run: rye sync

  ci:
    needs: setup
    uses: ./.github/workflows/ci.yml  # ci.yml呼び出し

  # 以下ジョブは割愛
```

ryeのセットアップは[setup-rye](https://github.com/eifinger/setup-rye)を使用します。
詳細は[Ryeとキャッシュ機能で快適なCI環境を整備してみた](https://zenn.dev/spectee/articles/spectee-rye-ci-cache)を参照ください。

https://zenn.dev/spectee/articles/spectee-rye-ci-cache

ポイントは2点
1点目は起動条件`on`キーに`pull_request`を追加することです。
プルリクエスト時にコメントする機能を実現するので必須となります。

```yml
on:
  push:
  pull_request:          # 重要:ワークフローの起動条件にプルリクエストを追加
```

2点目は権限設定`permissions`キーに適切な書き込み権限を付与することです。
`pull-requests`の権限を`write`にします。
カバレッジをプルリクエストに表示するだけであれば`content`の権限は`read`で大丈夫です。

```yml
permissions:
  pull-requests: write   # 重要:権限設定
  contents: read
```

次にci.ymlの実装を行います。

```yml:ci.yml
name: CI

on:
  workflow_call:

permissions:
  contents: write
  pull-requests: write

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: setup rye
        id: setup-rye
        uses: eifinger/setup-rye@v4
        with:
          enable-cache: true
          cache-prefix: rye-cache

      - name: install dependencies
        run: rye sync

      - name: run ci for python project # 重要:CIの実行（静的解析〜テスト）
        run: |
          rye run ci

      - name: comment coverage on pull request
        if: github.event_name == 'pull_request' # 重要:プルリクエストの時にテストカバレッジをコメントする
        uses: MishaKav/pytest-coverage-comment@main
        with:
          pytest-coverage-path: ./pytest-coverage.txt
          junitxml-path: ./pytest.xml
          title: Coverage Report（ServiceA）
          unique-id-for-comment: ServiceA # 同じコメントにテスト結果を追記するための一意なID
          create-new-comment: false # 既存のコメントを更新します。同じコメントにテスト結果を追記するため、新しいコメントは作成しません。
```

ポイントは2点。CI実施のステップとテストカバレッジのコメント処理です。

1点目はCIの実施です。このステップでpytest-coverage.txtとpytest.xmlが出力されます。

```yml
- name: run ci for python project
  run: |
    rye run ci
```

2点目はカバレッジのコメント処理です。
今回のワークフローはプルリクエスト以外にプッシュ時でも動作する仕組みになっています。
プルリクエストの時だけステップ処理を行いたいので`if`キーでイベントを制限します。

```yml
- name: comment coverage on pull request
  if: github.event_name == 'pull_request' # 重要:プルリクエストの時にテストカバレッジをコメントする
  uses: MishaKav/pytest-coverage-comment@main
  with:
    pytest-coverage-path: ./pytest-coverage.txt
    junitxml-path: ./pytest.xml
    title: Coverage Report（ServiceA）
    unique-id-for-comment: ServiceA 
    create-new-comment: false
```

公式サンプルからいくつかオプションを追加しています。

- `title`：コメントタイトル
- `unique-id-for-comment`: 同じコメントにテスト結果を追記するための一意なID。
1回のCIで復数のカバレッジを表示したいモノレポ構成の場合に指定しておくと、サービスごとにカバレッジコメントがでてきます
- `create-new-comment`: プルリクエスト更新時のコメント更新挙動です
  - true: 新しくコメントする
  - false: 既存のコメントを更新する

最終的なディレクトリ構成は以下の通りです。

```bash
.
└── .github
│   └── workflows
│       ├── ci.yml
│       └── cicd.yml
├── pyproject.toml
├── scripts
│   └── test_cov.sh
├── src              
└── tests  
```

### 手順3.動作確認

ワークフローの実装まで終わったらコードをプッシュしてプルリクエストを作成します。

プルリクエスト作成後にワークフローが動き始めます。
しばらくしてプルリクエストにカバレッジコメントが追加されたら成功です。

![PRコメント](https://storage.googleapis.com/zenn-user-upload/13501d449d4f-20241129.png)

`> Coverage Repost`のトグルを開くと未カバー部分を確認することができます。

![PRカバレッジ詳細](https://storage.googleapis.com/zenn-user-upload/cf65a0797611-20241129.png)

## まとめ

今回はpytestカバレッジをプルリクエストのコメントに表示する仕組みを紹介しました。
プルリクのコメントに表示されることでテストを確認する意識付けが向上したなと思いました。
レビューで開発プロセスの透明性と効率性を向上も期待できそうです！

## 参考

- [pytest-coverage-comment](https://github.com/marketplace/actions/pytest-coverage-comment)
