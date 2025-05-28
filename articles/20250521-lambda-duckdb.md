---
title: "【入門】AWS Lambda + Docker + DuckDBで作るデータ処理基盤 - Python 3.12対応"
emoji: "🐳"
type: "tech"
topics: ["lambda", "duckdb", "docker", "python", "aws"]
published: true
---

## 目次

1. [はじめに](#はじめに)
   - [前提条件](#前提条件)
   - [環境情報](#環境情報)
2. [アーキテクチャ概要](#アーキテクチャ概要)
   - [ベースイメージ](#ベースイメージ)
   - [ディレクトリ構成](#ディレクトリ構成)
3. [実装手順](#実装手順)
   - [Dockerfileの作成](#1-dockerfileを作成する)
   - [依存関係の定義](#2-requirementstxtに必要なライブラリを記載)
   - [Lambda関数の実装](#3-lambda-関数内で実行するpythonファイルを実装)
   - [ビルドとデプロイ](#4-docker-イメージのビルドと-ecr-への-push)
4. [運用とメンテナンス](#運用とメンテナンス)
   - [Lambda関数の更新方法](#lambda-関数の更新)
   - [モニタリングとロギング](#モニタリングとロギング)
   - [トラブルシューティング](#トラブルシューティング)
5. [参考リンクと関連リソース](#参考リンクと関連リソース)

## はじめに

この記事では、Dockerイメージを使用してAWS Lambda関数をデプロイする方法を解説します。
DuckDBを使用した実践的なデータ処理の例も含めて説明します。

### 前提条件
- AWS CLIのインストールと設定
- Dockerのインストール
- Python 3.12の基本的な知識
- AWS ECRの基本的な知識

### 環境情報
- OS: macOS/Linux
- Python: 3.12
- AWS CLI: 2.x
- Docker: 20.x以上

## アーキテクチャ概要

このセクションでは、実装するシステムの全体像について説明します。

### ベースイメージ

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
├── Dockerfile          # Dockerイメージの定義
├── lambda_function.py  # Lambda関数のメイン処理
├── requirements.txt    # 依存関係の定義
└── README.md          # プロジェクトの説明
```

## 実装手順

### 1. Dockerfileを作成する

下記を例に、Dockerfileを作成します。各コマンドの役割を説明するコメントを追加しています。

```docker: Dockerfile
FROM public.ecr.aws/lambda/python:3.12

# 依存関係のインストール
COPY requirements.txt ${LAMBDA_TASK_ROOT}
RUN pip install -r requirements.txt

# アプリケーションコードのコピー
COPY lambda_function.py ${LAMBDA_TASK_ROOT}

# Lambda関数のハンドラーを指定
CMD [ "lambda_function.lambda_handler" ]
```

### 2. requirements.txtに必要なライブラリを記載

requirements.txtにインストールするライブラリを記載します。
今回は、DuckDB（v0.10.1）と boto3（S3に接続する際に使用） をDLします。

```text
duckdb==0.10.1  # 高速な分析用データベース
boto3           # AWS SDK for Python
```

### 3. Lambda 関数内で実行するPythonファイルを実装

動作検証用に DuckDB を使用して、簡単な SQL クエリを実行する Lambda 関数を実装します。
型ヒントとエラーハンドリングを含めた実装例を示します。

> DuckDBとは：インメモリで高速な分析が可能なSQLデータベースエンジン  
> 詳細は[公式ドキュメント](https://duckdb.org/docs/stable/index)を参照

```python: lambda_function.py
import json
import logging
import duckdb
import boto3
from typing import List, Dict, Any

# ロガーの設定
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    try:
        con = duckdb.connect(database=':memory:')
        con.execute("SELECT 'hello' AS greeting, 123 AS number")
        result = con.fetchall()
        
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"data": result})
        }
    except Exception as e:
        logger.error(f"エラーが発生しました: {str(e)}")
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": str(e)})
        }

def process_data(con: duckdb.DuckDBPyConnection) -> List[Dict]:
    """サンプルデータの集計処理を行う関数
    
    Args:
        con: DuckDB接続オブジェクト
        
    Returns:
        集計結果のリスト
    """
    try:
        # サンプルデータの作成
        con.execute("""
            CREATE TABLE sales AS 
            SELECT * FROM (
                VALUES 
                ('2024-01-01', 1000),
                ('2024-01-02', 1500),
                ('2024-01-03', 2000)
            ) t(date, amount)
        """)
        
        # 集計クエリの実行
        con.execute("""
            SELECT 
                date,
                amount,
                SUM(amount) OVER (ORDER BY date) as cumulative_sum
            FROM sales
            ORDER BY date
        """)
        return con.fetchall()
    except Exception as e:
        logger.error(f"データ処理エラー: {str(e)}")
        raise
```

### 4. Docker イメージのビルドと ECR への Push

上記のファイルを作成したら、Docker イメージをビルドして、ECR に Push します。
事前に`test-lambda`という名前のリポジトリを ECR に作成しておく必要があります。

```bash
# プロジェクトルートに移動
cd lambda

# Dockerイメージのビルド（M1/M2 Macの場合）
docker buildx build --platform linux/amd64 --provenance=false -t test-lambda:latest .

# タグ付け
docker tag test-lambda:latest xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest

# ECRログイン
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin xxxx.dkr.ecr.ap-northeast-1.amazonaws.com

# ECRにPush
docker push xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest
```

## 運用とメンテナンス

### Lambda 関数の更新

Dockerイメージを更新した場合、Lambda 関数を更新する必要があります。
ECRにdocker pushしただけでは、古いイメージを参照してしまいます。

```bash
# Lambda関数の更新
aws lambda update-function-code \
  --function-name "test-lambda" \
  --image-uri "xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-lambda:latest"
```

### モニタリングとロギング

- CloudWatch Logsでのログ確認方法
- メトリクスの監視ポイント
- アラートの設定

### トラブルシューティング

よくある問題と解決方法：

1. **ビルドエラー**
   - M1/M2 Macでビルドする場合は`--platform linux/amd64`オプションを指定
   - 依存関係のインストールエラーはバージョンを確認

2. **実行時エラー**
   - メモリ不足：`memory_size`を増やす
   - タイムアウト：`timeout`値を調整
   - 権限エラー：IAMロールを確認

3. **デプロイエラー**
   - ECRプッシュ失敗：認証情報を確認
   - Lambda更新失敗：イメージURIを確認

## 参考リンクと関連リソース

### AWS公式ドキュメント
- [AWS Lambda Python Docker イメージ](https://gallery.ecr.aws/lambda/python)
- [AWS Lambda デベロッパーガイド](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-image.html)
- [Amazon ECR ユーザーガイド](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/what-is-ecr.html)

### 使用技術のドキュメント
- [DuckDB 公式ドキュメント](https://duckdb.org/docs/stable/index)
- [Docker ドキュメント](https://docs.docker.com/)
- [Python 公式ドキュメント](https://docs.python.org/3.12/)

### その他の参考資料
- [AWS Lambda のベストプラクティス](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/best-practices.html)
- [コンテナイメージのセキュリティベストプラクティス](https://docs.docker.com/develop/security-best-practices/)
- [AWS Well-Architected Framework](https://aws.amazon.com/jp/architecture/well-architected/)
