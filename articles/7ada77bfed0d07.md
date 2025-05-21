---
title: "FireLens × Fluent Bit によるログルーティング構成を用いてS3にログを送信する方法"
emoji: "🐥"
type: "tech"
topics:
  - "s3"
  - "ecs"
  - "firelens"
  - "ログ"
  - "fluentbit"
published: true
published_at: "2025-05-21 11:52"
---


今回は、FireLens と Fluent Bit によるログルーティングの方法やTipsについて書いていこうと思います。


---

## 🔍 FireLensとは？

**FireLens**は、AWS が提供する ECS タスク上で **FluentBit / Fluentd** を使ってログルーティングを行うための仕組みです。

ECSタスクに定義されているコンテナ群に、FireLens コンテナ を含めることで、アプリケーションコンテナから出力されたログを **S3, CloudWatch Logs などに転送**する事ができます。

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_firelens.html

## 🔍 FluentBitとは？

FluentBitは軽量で高速なログ収集・転送ツール。

さまざまな入力ソースからログを収集・処理し、保存前に解析・フィルタリングを行う。

https://docs.fluentbit.io/manual/stream-processing/overview#fluent-bit-data-pipeline




## ログルーティングの方法

### 1. FireLens の**デフォルト設定**でログを S3 に送る方法

- 前提
  - S3バケット「`test-fluentbit`」にアプリケーションのログを送る

- 主な実装のポイント
  - [AWSのpublicイメージ](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/firelens-using-fluentbit.html)を利用する
  - firelensコンテナ自体のログはawslogsを使ってCloudWatchに送る
  - FluentBitの設定ファイルはAWSのpublicイメージに置いてあるデフォルトの設定を利用する
      - → ここでは別途FluentBitの設定をする必要はない。

**ECSタスク定義の実装例：**
::::details ECSタスク定義内のファイル（firelensコンテナの定義）
```json
...
{
  "name": "firelens",
  "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:stable",
  "essential": true,
  "firelensConfiguration": {
    "type": "fluentbit"
  },
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-region": "ap-northeast-1",
      "awslogs-create-group": "true",
      "awslogs-group": "firelens"
    }
  }
},
...

```
::::


::::details ECSタスク定義内のファイル（アプリケーションコンテナの定義）
```json
{
  "name": "application",
	"logConfiguration": {
		  "logDriver": "awsfirelens",
		  "options": {
		    "Name": "s3",
		    "region": "ap-northeast-1",
		    "bucket": "test-fluentbit",
		    "use_put_object": "On",
		    "total_file_size": "1M"
		  }
	},
	...
```
::::

:::message
この時、 ECS タスクロールに S3バケットに対する`s3:PutObject` , `s3:ListBucket`権限を追加する必要があります
:::

::::details ECSタスクロールにアタッチするポリシーの例
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::test-fluentbit/*",
        "arn:aws:s3:::test-fluentbit"
      ]
    }
  ]
}

```
::::




---

### 2. **カスタム Fluent Bit 設定ファイル**を定義してログをルーティングする方法（DockerImageを用いる方法）

**カスタム Fluent Bit 設定ファイル**でログをルーティングする方法は２つあります

1. **カスタム Fluent Bit 設定ファイルを含めた**Docker Imageを作成し、それを用いてコンテナを生成する方法
2. S3に**カスタム Fluent Bit 設定ファイルを置き、firelensコンテナにマウントさせる方法**

ここでは1について解説します。

次のようなディレクトリ構成でDockerImage生成用のファイルを作成するとします。

```json
project-root/
├── log/
│   ├── fluent-bit-custom.conf    # FluentBit の出力先設定など
│   └── Dockerfile                # firelensコンテナイメージ生成用
```

#### 1. Fluent Bit の設定ファイルを作成する

- 「環境変数：`${S3_BUCKET}` 」に定義したS3バケットにログを送信する

> 環境変数(FluentBit公式ドキュメントより)
> https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml/environment-variables-section

```conf: fluent-bit-custom.conf
[OUTPUT]
    Name s3
    Match *
    region ap-northeast-1
    bucket ${S3_BUCKET}
    total_file_size 1M
    upload_timeout 1m
    use_put_object On

```

#### 2. Dockerfile を作成

```docker: Dockerfile
# aws公式のfirelensイメージを上書きする形でDockerImageを作成する
FROM public.ecr.aws/aws-observability/aws-for-fluent-bit:stable

# カスタム Fluent Bit 設定ファイルをDockerImage内にコピーする
COPY fluent-bit-custom.conf /fluent-bit/etc/fluent-bit-custom.conf
```

#### 3. DockerImageをビルドして ECR にプッシュ

- 事前にECRに`test-firelens` という名前のリポジトリを作成しておく必要があります。

```bash

# Docker Imageをビルドする
docker build -t xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-firelens:latest ./log/

# ECRにログインする
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin xxxx.dkr.ecr.ap-northeast-1.amazonaws.com

# ECRにDocker Imageをプッシュする
docker push xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-firelens:latest

```

#### 4. ECS タスク定義を更新

**FireLens コンテナ（カスタム設定読み込み）**
- 主な実装のポイント
  - image
    - 上記で生成したDockerImageのURIを参照する
  - firelensConfiguration > config-file-value > options
    - 上記で生成したDockerImage内の`fluent-bit-custom.conf`を参照する
  - S3_BUCKETには任意のS3バケット名を設定する
    - ここでは`test-fluentbit` が送信先となっている
  - アプリケーションコンテナの`logConfiguration`には出力先等の記載は不要
    - `fluent-bit-custom.conf`で出力先の設定をしているため。
      
::::details ECSタスク定義内のファイル（firelensコンテナの定義）
```json
{
  "name": "firelens",
  "image": "xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-firelens:latest",
  "firelensConfiguration": {
    "type": "fluentbit",
    "options": {
      "config-file-type": "file",
      "config-file-value": "/fluent-bit/etc/fluent-bit-custom.conf"
    }
  },
  "environment": [
    {"name": "S3_BUCKET", "value": "test-fluentbit"}
  ],
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-region": "ap-northeast-1",
      "awslogs-group": "firelens"
    }
  }
}

```
::::

::::details ECSタスク定義内のファイル（アプリケーションコンテナの定義）
```json
{
  "name": "application",
	"logConfiguration": {
		  "logDriver": "awsfirelens"
	},
	...
```
::::



## Parserの活用方法

> PARSER(FluentBit公式ドキュメントより)
> [https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml/environment-variables-section](https://docs.fluentbit.io/manual/pipeline/parsers/configuring-parser)

### 1. 既存のParserを利用する方法

firelensのイメージ内に既存のParserファイルがあるため、そちらを利用する。

> 既存のParser
> https://github.com/fluent/fluent-bit/blob/master/conf/parsers.conf


今回は、`nginx`のParserルールを使う。こちらのParserはすでに実装されているため、改めて実装する必要はない。
```
[PARSER]
    Name nginx
    Format regex
    Regex ^(?<remote>[^ ]*) - (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<request>[^"]*)" (?<code>\d*) (?<size>\d*)
    Time_Key time
    Time_Format %d/%b/%Y:%H:%M:%S %z

```

FluentBitの設定に下記を追加することで`nginx`のParserルールが機能する。
```conf: fluent-bit-custom.conf
[SERVICE]
    parsers_file /fluent-bit/etc/parsers.conf

[FILTER]
    Name parser
    Match *
    Key_name log
    Parser nginx

```

↓するとこのログが、

```docker
192.168.2.20 - - [29/Jul/2015:10:27:10 -0300] "GET /cgi-bin/try/ HTTP/1.0" 200 3395

```

↓こんな感じでParseされる
```json
{
  "host": "192.168.2.20",
  "user": "-",
  "time": "29/Jul/2015:10:27:10 -0300",
  "method": "GET",
  "path": "/cgi-bin/try/",
  "code": 200,
  "size": 3395,
  "referer": "",
  "agent": ""
}
```

---
### 2. 独自にParserルールを定義する方法

ここでは独自にParserルールを定義して、FluentBit内で利用する方法を書く。

↓このログを

```docker
ip: 192.168.2.20 request_time: 2015:10:27:10 user_id: 3395
```

↓こんな感じでパースしたい

```json
{
  "ip": "192.168.2.20",
  "request_time": "2015:10:27:10",
  "user_id": "3395"
}
```


`parsers-custom.conf`を新たに作成し、`fluent-bit-custom.conf`と同じ階層に置く。
今回は、`access_log`というルールを作成した。

```conf: parsers-custom.conf
[PARSER]
    Name   custom_log
    Format regex
    Regex  ip\:(?<ip>[^\]]+) request_time\:(?<request_time>[^\]]+) user_id\:(?<user_id>[^\]]+)
```

ちなみに、Parseのルールは下記サイトで簡単に試すことができる。
https://rubular.com/r/X7BH0M4Ivm

`fluent-bit-custom.conf`も下記のように実装する。

```conf: fluent-bit-custom.conf
[SERVICE]
    parsers_file /fluent-bit/etc/parsers-custom.conf

[FILTER]
    Name parser
    Match *
    Key_name log
    Parser custom_log

```

`parsers-custom.conf`と、`fluent-bit-custom.conf`を実装したら、`Dockerfile`を次のように書き換えてfirelensのコンテナイメージをデプロイした後、ECSタスクを置き換える。

```docker: Dockerfile
# aws公式のfirelensイメージを上書きする形でDockerImageを作成する
FROM public.ecr.aws/aws-observability/aws-for-fluent-bit:stable

# カスタム Fluent Bit 設定ファイルをDockerImage内にコピーする
COPY fluent-bit-custom.conf /fluent-bit/etc/fluent-bit-custom.conf

# カスタム ParserファイルをDockerImage内にコピーする
COPY parsers-custom.conf /fluent-bit/etc/parsers-custom.conf
```


## 便利なFilter

### RewriteTag

特定のログのみ別の場所に送りたい時とかに有効。

> Rewrite Tag(FluentBit公式ドキュメントより)
> https://docs.fluentbit.io/manual/pipeline/filters/rewrite-tag


例）

「ERROR」を含むログは別のS3バケットに送りたい。

```conf: fluent-bit-custom.conf
[FILTER]
    Name rewrite_tag
    Match *-firelens-*
    Rule $log ERROR error false
    
[OUTPUT]
    Name s3
    Match error
    region ap-northeast-1
    bucket error-logs
    total_file_size 1M
    upload_timeout 1m
    use_put_object On
```

補足：

FireLensを用いる場合、FireLens独自の仕様で、初めからログに対して以下の形式のタグが自動的に付与される。

```
<container name>-firelens-<task ID>
```

参考：
https://dev.classmethod.jp/articles/ecs_firelens_tag/

そのため、アプリケーションのログを指定するには`*-firelens-*`とすれば良さそう

---

### Record Modifier
送信するログの要素（date, user_id）などを絞りたい時などに有効。

> Record Modifier(FluentBit公式ドキュメントより)
> https://docs.fluentbit.io/manual/pipeline/filters/record-modifier


例）

下記のようなログがある。この時、「host, time, method, pathのみ」をS3に送りたい

```json
{
  "host": "192.168.2.20",
  "user": "-",
  "time": "29/Jul/2015:10:27:10 -0300",
  "method": "GET",
  "path": "/cgi-bin/try/",
  "code": 200,
  "size": 3395,
  "referer": "",
  "agent": ""
}
```

このように設定ファイルを実装することで、出力項目を絞ったり追加したりできる。
```conf: fluent-bit-custom.conf
[FILTER]
    Name record_modifier
    Match *-firelens-*
    Allowlist_key host
    Allowlist_key time
    Allowlist_key method
    Allowlist_key path
```

↓実際に送られるログは下記のようになる。
```json
{
  "host": "192.168.2.20",
  "time": "29/Jul/2015:10:27:10 -0300",
  "method": "GET",
  "path": "/cgi-bin/try/"
}
```


## ⚠️ Fargate上での注意点

- Fargateなどの永続ディスクのない環境で実行する場合は設定に注意が必要です
- 下記参照（ざっくりと、細かくログを送りましょうみたいな話です）

https://docs.fluentbit.io/manual/pipeline/outputs/s3#using-s3-without-persisted-disk


## 関連ページ

https://aws.amazon.com/jp/blogs/news/under-the-hood-firelens-for-amazon-ecs-tasks/

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_firelens.html

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/firelens-taskdef.html

https://docs.fluentbit.io/manual/about/what-is-fluent-bit

https://rubular.com/r/X7BH0M4Ivm