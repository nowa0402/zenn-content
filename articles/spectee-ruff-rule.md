---
title: "チームのRuff設定を見直してみた"
emoji: "🔍️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "ruff"]
published: false
publication_name: "spectee"
---

## はじめに・導入経緯

Specteeでエンジニアをしている和山と申します！
私の所属するチームでは、Pythonの静的解析及びフォーマットにRuffを採用しています。

https://docs.astral.sh/ruff/

チームへ導入を始めて約1年経つため、メンテナンス含めて設定の見直しを行いました！

## 本記事の結論

本記事の最終的な設定です。

```toml:pyproject.toml
[tool.ruff]
# 指定されたPythonバージョンで構文チェックを行う(主にpyupgrade)
target-version = "py312" # Python >=3.12 or ==3.12


lint.select = [
    "F",  # pyflakes
    "E",  # pycodestyle
    "W",  # pycodestyle warnings
    "I",  # isort
    "D",  # pydocstyle
    "UP", # pyupgrade
    "N",  # pep8-naming
]

lint.ignore = [
    "D105", # undocumented-magic-method
    "D107", # undocumented-public-init
    "D205", # blank-line-after-summary
    "E402", # sys path before import
    "D415", # ends-in-punctuation
]

line-length = 100

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401", "D"] # __init__.pyは未使用インポートとドキュメントを無視
```

```json:.vscode/settings.json
{
    // pythonの設定だけ抜粋
    "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  }
}
```

## 現状の設定

まずは、現状の設定です。
`pyproject.toml`ファイルにRuffのルールを記載しています。

```toml:pyproject.toml
[tool.ruff]
lint.select = [
    "F", # pyflakes
    "E", # pycodestyle
    "W", # pycodestyle warnings
    "I", # isort
    "D", # pydocstyle
]
lint.ignore = [
    "D105", # undocumented-magic-method
    "D107", # undocumented-public-init
    "D205", # blank-line-after-summary
    "E402", # sys path before import
    "D415", # ends-in-punctuation
]

line-length = 100

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401", "D"] # __init__.pyは未使用インポートとドキュメントを無視
```

ルールの詳細は[公式サイト](https://docs.astral.sh/ruff/rules/)参照
簡単な概要を👇️にまとめます。

| ルール名  | 略称 | 説明 |
| ------------- | ------------- | ----- |
| Pyflakes  | F  | 未使用インポートや未定義変数のチェック |
| pycodestyle  | E  | PEP8準拠のコードスタイル違反のチェック |
| pydocestyle warning  | W | pycodestyleの派生 |
| isort  | I | インポート順序のチェック・並び替え |
| pydocstyle  | D  | docstringのチェック |

その他特出点は以下の通り

- `convention = "google"`でdocstringの形式をGoogleStyleに設定
- `__init__.py`はモジュールエクスポートのみで使用しているため、個別パターンを適用
- `line-length = 100`で改行の文字数を指定※

:::message
※ 文字数は[PEP8](https://pep8-ja.readthedocs.io/ja/latest/#id8)準拠だと79文字以内です。
チーム事情により100文字としています。
:::

開発ではVsCodeを使っているため、`.vscode/settings.json`も設定します。

```json:.vscode/settings.json
{
    // pythonの設定だけ抜粋
    "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  }
}
```

上記設定で動かすにはRuffの拡張機能が必須です。

https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff

## 静的解析の目的は「プロジェクトの持続可能性を高める」こと

見直しに入る前に…。最初に静的解析の目的を整理します。

静的解析に何を期待するかはチームによって異なると思います。
ここでは、この記事における目的を定義したいと思います。

まずは抽象レベルが高いところの定義。静的解析が目指す最終目的です。
これは **「プロジェクトの持続可能性を高める」** と定義します。

例として、書籍：[単体テストの考え方/使い方](https://www.amazon.co.jp/%E5%8D%98%E4%BD%93%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9-%E4%BD%BF%E3%81%84%E6%96%B9-Vladimir-Khorikov/dp/4839981728)では、単体テストの目的を以下のように定義しています。

> 1.2 なぜ単体テストを行うのか？
> 単体テストをすることで何を成し遂げたいのでしょうか？その答えは、ソフトウェア開発プロジェクトの成長を持続可能なものにする、ということです。

https://www.amazon.co.jp/%E5%8D%98%E4%BD%93%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9-%E4%BD%BF%E3%81%84%E6%96%B9-Vladimir-Khorikov/dp/4839981728

もう一例、書籍：[GitHub CI/CD実践ガイド](https://gihyo.jp/book/2024/978-4-297-14173-8)では、CI/CDの目的を以下のように定義しています。

> 1.2 CI/CD
> CI/CDの目的は次のようにまとめられます。
> ソフトウェア開発の持続可能性を高め、長期に渡る価値提供を実現する

https://gihyo.jp/book/2024/978-4-297-14173-8

両例ともに **「持続可能性」** を目的に掲げており
そのアプローチの一つとして「静的解析」も同枠に存在していると考え、最終目的を定義しました。

次に静的解析がプロジェクトの持続可能性を高めるためにできる具体的な目的です。
これは **「開発者全員がコード品質を一定に保つための仕組み」** と定義します。

ソフトウェア開発におけるコードは常に一定の品質で保たれていることで、開発者は本質的な作業に集中することができ、プロジェクトの持続的成長を促すことができると考えます。そのための取り組みの一つとして「静的解析」を行います。
👇️は静的解析によって得られる効果の一例です。

- バグやエラーの早期発見に役立てる
  - スキルが未成熟である開発者でも、一定の品質を担保したコードが書けるようになる
  - コードレビュー時にレビュアーが些細なミスを指摘せずに済む（本質的な変更箇所に時間をかけられる）
- セキュリティの強化。未然に脆弱性をスキャンし対応することが可能になる

これらを達成する仕組みの整備と見直しを行っていくのが本記事になります。

## 見直しポイント

静的解析で達成したい目的と現状の設定を踏まえ、現状の設定から見直しポイントは以下の通りです。

- 変数やクラス名の命名規則の誤りを検知したい
- 古いPythonバージョンの書き方を検知したい

いずれも、設定することでコーディング品質を保つ目的に貢献すると考えています

### 変数やクラス名の命名規則の誤りを検知したい

私たちのチームでは、Pythonの他にTypeScriptを用いた開発を行います。
複数言語を行き来すると、PythonのコードにTypeScriptの命名規則で書いてしまう場面がありました。

例えば、Pythonの変数名はスネークケースです。

```python
user_name = "Bob"
```

一方、TypeScriptの変数名はキャメルケースです。

```typescript
const userName = "Bob"
```

このような命名規則による誤りを減らすために、Ruffの設定に存在する[pep8-naming](https://docs.astral.sh/ruff/rules/#pep8-naming-n)を採用することにしました。
pep8-namingを使用すれば、クラスや変数名の命名規則を[PEP8](https://pep8-ja.readthedocs.io/ja/latest/#id23)に準拠した書き方に統一できます。

命名規則に統一感がないとコードを読む時にノイズを感じてしまうため、お手軽にかつ高い効果を得られると思います。

設定は以下の通り。`N`を追加します。

```toml:pyproject.toml
[tool.ruff]
lint.select = [
    "N",  # pep8-naming
]
```

### 古いPythonバージョンの書き方を検知したい

Pythonは言語自体がそれなりの頻度でバージョンアップされます。
バージョンアップに伴い、コーディングで非推奨になった書き方がいくつか存在します。

例として、`Optional`や`Union`型の指定があります。
Python3.9までは複数の型の指定は標準ライブラリ`typing`からインポートする必要がありました。

```python
from typing import Optional


def to_number(s: str) -> Optional[int]:
    try:
        return int(s)
    except ValueError:
        return None
```

Python3.10以降は`|`で表現できます。

```python
def to_number(s: str) -> int | None:
    try:
        return int(s)
    except ValueError:
        return None
```

非推奨になったコードを放置し続けると、あるバージョンからサポートが切られてしまい不具合につながる可能性があります。
これを出来るだけ早い段階で検知するため、[pyupgrade](https://docs.astral.sh/ruff/rules/#pyupgrade-up)を採用します。

設定は以下の通り。
Pythonのバージョン指定の設定（[target-version](https://docs.astral.sh/ruff/settings/#target-version)）が必要になります。

なお、バージョンは`[project]`もしくは`[tool.ruff]`に指定します。
どちらにも書いた場合は`[tool.ruff]`の`target-version`が優先されます。

```toml:pyproject.toml
#[project]
#requires-python = ">=3.12"

[tool.ruff]
# 指定されたPythonバージョンで構文チェックを行う
target-version = "py312" # Python >=3.12 or ==3.12 という意味

lint.select = [
    "UP", # pyupgrade
]
```

## 見直し後の設定

見直しを含めて整理した`pyproject.toml`はこちら

```toml:pyproject.toml
[tool.ruff]
# 指定されたPythonバージョンで構文チェックを行う(主にpyupgrade)
target-version = "py312" # Python >=3.12 or ==3.12


lint.select = [
    "F",  # pyflakes
    "E",  # pycodestyle
    "W",  # pycodestyle warnings
    "I",  # isort
    "D",  # pydocstyle
    "UP", # pyupgrade
    "N",  # pep8-naming
]

lint.ignore = [
    "D105", # undocumented-magic-method
    "D107", # undocumented-public-init
    "D205", # blank-line-after-summary
    "E402", # sys path before import
    "D415", # ends-in-punctuation
]

line-length = 100

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401", "D"] # __init__.pyは未使用インポートとドキュメントを無視
```

ルールの一言説明は以下の通り

| ルール名  | 略称 | 説明 |
| ------------- | ------------- | ----- |
| Pyflakes  | F  | 未使用インポートや未定義変数のチェック |
| pycodestyle  | E  | PEP8準拠のコードスタイル違反のチェック |
| pydocestyle warning  | W  | pycodestyleの派生 |
| pydocstyle  | D  | docstringのチェック |
| isort  | I | インポート順序のチェック・並び替え |
| pyupgrade  | UP |  Pythonコードが最新バージョンに対応しているかチェック |
| pep8-naming  | N | Pythonコードの命名規則がPEP8に準拠しているかチェック |

開発ではVsCodeの設定は変更なしです。

```json:.vscode/settings.json
{
    // pythonの設定だけ抜粋
    "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit"
    }
  }
}
```

## まとめ

今回はRuffの設定見直しを行いました。見直す上で、改めてRuffのドキュメントを眺めていると最初設定した頃から随分増えたなあ…という印象がありました。

今後もRuffの動向を気にしつつ、定期的にメンテナンスを行い
チーム全員がプロダクト開発に集中できる環境を用意したいと思います！

## 参考

- [Rules | Ruff](https://docs.astral.sh/ruff/rules/)
- [単体テストの考え方/使い方](https://www.amazon.co.jp/%E5%8D%98%E4%BD%93%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9-%E4%BD%BF%E3%81%84%E6%96%B9-Vladimir-Khorikov/dp/4839981728)
- [GitHub CI/CD実践ガイド](https://gihyo.jp/book/2024/978-4-297-14173-8)
- [pep8-ja](https://pep8-ja.readthedocs.io/ja/latest/)
