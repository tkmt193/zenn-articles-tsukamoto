---
title: ""
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

以下は、**Docker イメージを使って AWS Lambda 関数を構築・デプロイ・更新・実行する一連の手順**をまとめた技術記事のテンプレートです。必要に応じて Notion や Zenn に貼り付けられる構成になっています。

---

# Lambda × Docker × DuckDB を用いたログ抽出基盤の構築メモ

## 概要

このドキュメントでは、以下の構成で AWS Lambda 関数を構築・デプロイ・実行するワークフローについて記載します。

* Python + DuckDB を含む Lambda 関数の Dockerfile 作成
* ECR への Docker イメージ Push
* Terraform による Lambda 関数定義
* CLI 経由での Lambda 更新・Invoke（実行）

---

## 1. Lambda 関数（Docker イメージ構成）

### ベースイメージ

```text
public.ecr.aws/lambda/python:3.12
```

AWS Lambda 用の公式 Python 3.12 イメージをベースにします。

### ディレクトリ構成（例）

```
lambda/
├── Dockerfile
├── lambda_function.py
├── requirements.txt
└── event.json
```

---

## 2. Dockerfile の作成

```dockerfile
FROM public.ecr.aws/lambda/python:3.12

# 依存ライブラリをインストール
COPY requirements.txt ${LAMBDA_TASK_ROOT}
RUN pip install -r requirements.txt

# Lambda関数本体をコピー
COPY lambda_function.py ${LAMBDA_TASK_ROOT}

# エントリポイント
CMD [ "lambda_function.lambda_handler" ]
```

---

## 3. requirements.txt

```text
duckdb==0.10.1
boto3
```

---

## 4. Lambda 関数のダミーコード（lambda\_function.py）

```python
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

---

## 5. Docker イメージのビルドと ECR への Push

```bash
# 変数定義
export ECR_REPO=mikata-access-lambda
export AWS_ACCOUNT_ID=502221051993
export REGION=ap-northeast-1

# Dockerイメージのビルド
docker buildx build --platform linux/amd64 --provenance=false -t ${ECR_REPO}:latest .

# タグ付け
docker tag ${ECR_REPO}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO}:latest

# ECRログイン
aws ecr get-login-password --region ${REGION} | \
  docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# Push
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO}:latest
```

---

## 6. Lambda 関数の Terraform 定義（抜粋）

```hcl
resource "aws_lambda_function" "mikata_access_logs" {
  function_name = "mikata_access_logs_reports-${var.environment}"
  package_type  = "Image"
  image_uri     = "${var.aws_account_id}.dkr.ecr.${var.region}.amazonaws.com/${var.ecr_repo}:latest"
  role          = aws_iam_role.lambda_exec.arn
  timeout       = 300
  memory_size   = 512
}
```

---

## 7. Lambda 関数の更新

```bash
aws lambda update-function-code \
  --function-name "mikata_access_logs_reports-${ENVIRONMENT_NAME}" \
  --image-uri "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO}:latest"
```

---

## 8. Lambda 関数の実行（Invoke）

```bash
aws lambda invoke \
  --function-name mikata_access_logs_reports-${ENVIRONMENT_NAME} \
  --cli-binary-format raw-in-base64-out \
  --payload file://event.json \
  /dev/stdout | jq
```

`event.json` のサンプル：

```json
{
  "company_id": "1690",
  "start_date": "2025-04-01",
  "end_date": "2025-04-30"
}
```

---

## おわりに

DuckDB は Lambda のメモリDBとして扱えるので、データ加工・抽出処理に非常に便利です。`httpfs` を組み合わせれば S3 上のログを直接読み込むことも可能です。

Terraform による Lambda 関数定義・デプロイパイプラインと合わせて、再利用性の高いログ抽出基盤を実現できます。

---

必要に応じてこのテンプレートを Notion にコピペして使えます。さらに拡張したい場合（例：CloudWatch ログ、リトライ処理など）も対応できますので、お気軽にどうぞ。
