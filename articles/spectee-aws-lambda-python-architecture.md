---
title: "PythonでAWS LambdaをArm版にする方法：GitHubActions×CDKでバンドル時のイメージがx86になる問題を解決"
emoji: "👟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "aws", "cdk", "lambda", "githubactions"]
published: false
publication_name: "spectee"
---

Specteeでエンジニアをしている和山と申します！

PythonでAWS LambdaをCDKを使ってArm版にしたいと思ったことはありませんか？
Arm版はx86版より[料金が安く設定](https://aws.amazon.com/jp/lambda/pricing/)されているのと、x86版より若干性能が良いため経済的にLambdaを利用することができます。
少し古めの記事ですが、料金と性能比較をしてくださっている記事があったので紹介します。
https://dev.classmethod.jp/articles/aws-lambda-graviton2/

このように経済的なArm版のLambdaですが、CDKで実装する際にArm版として実装したのにx86版としてバンドルされてしまうことがあります。本記事ではこの問題の原因と、正しいバンドル方法について解説します。

## 本記事の結論

PythonでArm版のLambdaを指定するときは下記の実装をします。
ポイントは`code.bundling.image`にarm64版のイメージを明示的に指定することです。

```ts:cdk/lib/cdk-stack.ts
import * as cdk from 'aws-cdk-lib';
import { aws_lambda as lambda } from 'aws-cdk-lib';
import type { Construct } from 'constructs';
import * as path from 'node:path';

export class CdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const dirPath = path.resolve(__dirname, '..', '..');

    new lambda.Function(this, 'SamplePythonArmLambda', {
      functionName: 'sample-python-arm64-lambda',
      runtime: lambda.Runtime.PYTHON_3_11,
      architecture: lambda.Architecture.ARM_64,
      handler: 'main.handler',
      code: lambda.Code.fromAsset(dirPath, {
        // バンドルの設定を追加
        bundling: {
          // イメージにarm64を指定
          image: new lambda.Runtime(
            'python3.11:latest-arm64',
            lambda.RuntimeFamily.PYTHON,
          ).bundlingImage,
          command: [
            'bash',
            '-c',
            'pip install -r requirements.txt -t /asset-output && cp -au src/* /asset-output',
          ],
        },
      }),
    });
  }
}

```

## 構成

今回の構成です。

```bash
.
├── .github
│   └── workflows
│       └── cd.yml
├── cdk
│   ├── bin
│   │   └── cdk.ts
│   └── lib
│       └── cdk-stack.ts
├── requirements.txt
└── src
    └── main.py
```

`src/main.py`はpydanticをインポートして適当な値を返すLambdaを作成します。

```python:src/main.py
"""Main module"""

from typing import Any

from pydantic import BaseModel


class User(BaseModel):
    """User model"""

    id: int
    name: str
    age: int


def handler(event: dict[str, Any], context: dict[str, Any]) -> dict[str, Any]:
    """Lambda handler"""
    user = User(id=1, name="John", age=30)
    return user.model_dump()
```

`requirements.txt`は依存関係を記載しています。

```txt:requirements.txt
annotated-types==0.7.0
pydantic==2.10.5
pydantic-core==2.27.2
typing-extensions==4.12.2
```

`cdk/lib/cdk-stack.ts`はArm版のLambda設定を記載します。
バンドルの設定は[公式ドキュメント](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda-readme.html#bundling-asset-code)の設定を参考にしています。

```ts:cdk/lib/cdk-stack.ts
import * as cdk from 'aws-cdk-lib';
import { aws_lambda as lambda } from 'aws-cdk-lib';
import type { Construct } from 'constructs';
import * as path from 'node:path';

export class CdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const dirPath = path.resolve(__dirname, '..', '..');

    new lambda.Function(this, 'SamplePythonArmLambda', {
      functionName: 'sample-python-arm64-lambda',
      runtime: lambda.Runtime.PYTHON_3_11,
      architecture: lambda.Architecture.ARM_64,
      handler: 'main.handler',
      code: lambda.Code.fromAsset(dirPath, {
        // バンドルの設定を追加
        bundling: {
          platform: 'linux/arm64',
          image: lambda.Runtime.PYTHON_3_11.bundlingImage,
          command: [
            'bash',
            '-c',
            'pip install -r requirements.txt -t /asset-output && cp -au src/* /asset-output',
          ],
        },
      }),
    });
  }
}
```

`cdk/bin/cdk.ts`はStackをインポートして使用するだけです。
`cdk init`コマンドで作られたデフォルトのままでいけます。

```ts:cdk/bin/cdk.ts
import * as cdk from 'aws-cdk-lib';
import 'source-map-support/register';
import { CdkStack } from '../lib/cdk-stack';

const app = new cdk.App();
new CdkStack(app, 'SamplePythonArmStack', {});
```

`.github/workflows/cd.yml`にデプロイのワークフローを記載します。
今回は手動起動で動作するようにしています。ジョブのOSはubuntu(x86)なのでArm版のビルドを可能にするため[docker/setup-qemu-action](https://github.com/docker/setup-qemu-action)を採用しています。

```yml:.github/workflows/cd.yml
name: Deploy to SampleLambda

on:
  # GitHub Actions画面から手動で実行する
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      # マルチプラットフォームビルドのセットアップ
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
        with:
          platforms: all

      - name: install node
        uses: actions/setup-node@v4
        with:
          node-version: 20.x

      - name: Install node modules
        run: |
          cd cdk
          npm ci

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: "******" # ご自身の環境に合わせて修正してください
          aws-region: "ap-northeast-1"

      - name: Deploy
        run: |
          cd cdk
          npx cdk deploy --require-approval never
```

## Lambdaを実行するとエラーになる

GitHub ActionsでデプロイされたLambdaを実行すると次のエラーメッセージが返ってきます。

```json
{
  "errorMessage": "Unable to import module 'main': No module named 'pydantic_core._pydantic_core'",
  "errorType": "Runtime.ImportModuleError",
  "requestId": "",
  "stackTrace": []
}
```

pydanticのライブラリが読み込めていないようです。
Lambdaのコンソール画面を見ると、ライブラリ自体は存在しています。

![lambdaのコンソール画面](https://storage.googleapis.com/zenn-user-upload/c8b140b29a24-20250116.png)

### 原因はバンドル時のイメージだった

原因を調査していたら、バンドル時に使用しているイメージに不備があることが分かりました。
GitHub Actionsのログを見ていたらpydanticのライブラリインストールのアーキテクチャがx86になっていました。

```bash
Unable to find image 'public.ecr.aws/sam/build-python3.11:latest' locally
# ...
Collecting pydantic-core==2.27.2 (from -r requirements.txt (line 3))
  Downloading pydantic_core-2.27.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl.metadata (6.6 kB)
```

### cdkのオプションが反映されない模様

問題がCDKの設定と実際のイメージに差異があることだと分かりました。
おそらくCDK側の問題だろうと思いIssueを検索したら、２つほどヒットしました。

https://github.com/aws/aws-cdk/issues/18696
https://github.com/aws/aws-cdk/issues/30748

ざっくり要約すると **「platformの設定が反映されていない」** みたいです。
この場合はホストOSのプラットフォームを採用しているようです。今回はGitHub Actionsのubuntuを採用しているのでx86になったというわけです。

```ts
bundling: {
    platform: 'linux/arm64', // これが反映されない
    image: lambda.Runtime.PYTHON_3_11.bundlingImage,
    command: [
        'bash',
        '-c',
        'pip install -r requirements.txt -t /asset-output && cp -au src/* /asset-output',
    ],
},
```

## 修正方法

Issueの中で示されていた解決方法は２つありました。
１つ目は、「image」オプションに使うイメージでArm版を明示的に指定することです。
https://github.com/aws/aws-cdk/issues/18696#issuecomment-1027322761

コメントの内容を反映すると下記のようなコードになります。

```ts:cdk/lib/cdk-stack.ts
import * as cdk from 'aws-cdk-lib';
import { aws_lambda as lambda } from 'aws-cdk-lib';
import type { Construct } from 'constructs';
import * as path from 'node:path';

export class CdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const dirPath = path.resolve(__dirname, '..', '..');

    new lambda.Function(this, 'SamplePythonArmLambda', {
      functionName: 'sample-python-arm64-lambda',
      runtime: lambda.Runtime.PYTHON_3_11,
      architecture: lambda.Architecture.ARM_64,
      handler: 'main.handler',
      code: lambda.Code.fromAsset(dirPath, {
        // バンドルの設定を追加
        bundling: {
          // イメージにarm64を指定
          image: new lambda.Runtime(
            'python3.11:latest-arm64',
            lambda.RuntimeFamily.PYTHON,
          ).bundlingImage,
          command: [
            'bash',
            '-c',
            'pip install -r requirements.txt -t /asset-output && cp -au src/* /asset-output',
          ],
        },
      }),
    });
  }
}

```

`python3.11:latest-arm64`でarm64版のイメージを指定します。
ここで使用するタグは[sam/build-**](https://gallery.ecr.aws/sam)のイメージタグになります。

変更したものでデプロイをします。
その後テストすると、正常にレスポンスが返ってくることを確認できました！

```json
{
  "id": 1,
  "name": "John",
  "age": 30
}
```

GitHub Actionsのログをみてみると、使用イメージとpydanticライブラリがarm64版になっていることが確認できます

```bash
Status: Downloaded newer image for public.ecr.aws/sam/build-python3.11:latest-arm64
# ...
Collecting pydantic-core==2.27.2 (from -r requirements.txt (line 3))
  Downloading pydantic_core-2.27.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl.metadata (6.6 kB)
```

２つ目の方法は、`@aws-cdk/aws-lambda-python-alpha`を使用することです
https://docs.aws.amazon.com/cdk/api/v2/docs/aws-lambda-python-alpha-readme.html

[PIP_PLATFORMの環境変数に使用するプラットフォームを指定する](https://github.com/aws/aws-cdk/issues/18696#issuecomment-2196811267)ようです。

:::message alert
@aws-cdk/aws-lambda-python-alphaは2025/01/16時点では検証段階のため、
今後破壊的変更が入りコード例の通りに実装できない可能性があります
:::

Issueのコメントを抜粋したものが下記のコードになります

```ts
 new python_alpha.PythonFunction(this, "MyFunc", {
      entry: backendPath(
        "lambda/code/directory"
      ),
      runtime: Runtime.PYTHON_3_11,
      bundling: {
        buildArgs: {
          PIP_PLATFORM: "linux_arm64",
        },
      },
    });
```

## まとめ

今回はPythonでAWS LambdaをCDKを使ってArm版の指定時に躓いた点と解消方法を解説しました。

バンドル時の注意点をまとめると

- `code.bundling.platform`オプションは2025/01時点で反映されないっぽい
- `code.bundling.image`オプションで明示的にarm版のイメージタグを指定する

になります。

この記事が誰かのお役に立てば幸いです！
