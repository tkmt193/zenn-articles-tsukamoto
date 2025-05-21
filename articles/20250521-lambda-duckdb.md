---
title: "Dockerイメージを使って、AWS Lambda 関数（Python 3.12）をデプロイ・実行する方法"
emoji: "🐳"
type: "tech"
topics: ["lambda", "duckdb", "docker", "python"]
published: true
---

## 概要

このドキュメントでは、以下の構成で Dockerイメージを使って、AWS Lambda 関数を構築・デプロイ・実行するワークフローの実装方法について記載します。

### 元となるベースイメージ

今回、使用するランタイムは Python 3.12 とします。
その他の言語でも、同様の手順で実装可能です。

次の[AWS Lambda 用の公式 Python 3.12 イメージ](https://gallery.ecr.aws/lambda/python)をベースにLambda関数を構築して行きます。

```text
public.ecr.aws/lambda/python:3.12
```


### ディレクトリ構成

下記のようなディレクトリ構成を想定しています。

```
lambda/
├── Dockerfile
├── lambda_function.py
├── requirements.txt
...
```

## Docker イメージをビルドするための準備

### 1. Dockerfileを作成する

下記を例に、Dockerfileを作成します。

```docker: Dockerfile
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt ${LAMBDA_TASK_ROOT}
RUN pip install -r requirements.txt

COPY lambda_function.py ${LAMBDA_TASK_ROOT}

CMD [ "lambda_function.lambda_handler" ]
```


### 2. requirements.txtに必要なライブラリを記載

requirements.txtにインストールするライブラリを記載します。
今回は、DuckDB（v0.10.1）と boto3（S3に接続する際に使用） をDLします。

```text
duckdb==0.10.1
boto3
```


### 3. Lambda 関数内で実行するPythonファイルを実装（lambda\_function.py）

動作検証用に DuckDB を使用して、簡単な SQL クエリを実行する Lambda 関数を実装します。

> DuckDBとは
> https://duckdb.org/docs/stable/index

```python: lambda_function.py
import json
import duckdb
import boto3

def lambda_handler(event, context):
    con = duckdb.connect(database=':memory:')
    con.execute("SELECT 'hello' AS greeting, 123 AS number")
    result = con.fetchall()
    
    return {
        "statusCode": 200,
        "body": json.dumps(result)
    }
```



### 4. Docker イメージのビルドと ECR への Push

上記のファイルを作成したら、Docker イメージをビルドして、ECR に Push します。
事前に`test-lambda`という名前のリポジトリを ECR に作成しておく必要があります。

```bash
# プロジェクトルートに移動
cd lambda

# Dockerイメージのビルド
docker buildx build --platform linux/amd64 --provenance=false -t test-lambda:latest .

# タグ付け
docker tag test-lambda:latest xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest

# ECRログイン
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin xxxx.dkr.ecr.ap-northeast-1.amazonaws.com

# ECRにPush
docker push xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest
```



### 5. TerraformでLambda 関数を作成する

先ほど作成した Docker イメージを用いて、Terraform で Lambda 関数を作成します。
ここではTerraformのコードの実装例を記載しますが、Terraformの実行方法については、[Terraform公式ドキュメント](https://developer.hashicorp.com/terraform/docs/cli)を参照してください。

```hcl: lambda_sample.tf
resource "aws_lambda_function""test_lambda" {
  function_name = "test-lambda"
  package_type  = "Image"
  image_uri     = "xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest" # image_uri に 先ほどECRプッシュしたイメージの URI を指定
  role          = "arn:aws:iam::xxxx:role/lambda-execution-role"
  timeout       = 300
  memory_size   = 512
  ...
}
```

Lambda関数の詳しい書き方は下記を参照してください。
> aws_lambda_function (Terraform公式ドキュメント)
> https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function.html



## Lambda 関数の更新

Dockerイメージを更新した場合、Lambda 関数を更新する必要があります。
ECRにdocker pushしただけでは、古いイメージを参照してしまいます。（[AWS Lambda デベロッパーガイド](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-image.html#python-image-clients)）

AWS CLI を使用して Lambda 関数を更新する場合、以下のコマンドを実行します。

```bash
aws lambda update-function-code \
  --function-name "test-lambda" \
  --image-uri "xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest"
```



## Lambda 関数の実行（Invoke）

AWS CLI を使用して Lambda 関数を更新する場合、以下のコマンドを実行します。

```bash
aws lambda invoke \
  --function-name "test-lambda" \
  --cli-binary-format raw-in-base64-out \
  --payload  '{"payload":"hello world!"}'
```

## ローカルでLambda 関数を実行する

docker buildx build でビルドしたイメージは、ローカルでも実行可能です。

ローカルで Lambda 関数を実行する場合、まず以下のコマンドを実行して、Docker イメージを起動します。

```bash
# Docker イメージを起動
docker run --platform linux/amd64 -p 9000:8080 test-lambda:latest
```

別のセッションで以下のコマンドを実行し、Lambda 関数を実行します。
```bash
# Lambda 関数を実行
curl "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{"payload":"hello world!"}'
```


## 関連ページ
https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-image.html#python-image-instructions
