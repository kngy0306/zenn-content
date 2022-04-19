---
title: "DockerでServerlessを実行できる環境の作成"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "serverless"]
published: true
---

ローカル環境にServerlessをインストールし開発を行っていましたが、Apple SiliconのPCでは使用したいプラグイン(`serverless-dynamodb-local`)が対応していなかったため、Dockerを用いてServerlessの環境を作成しました。

前提として、AWSのアカウントが作成済みで、AWSへデプロイすることを想定しています。

# Dockerfile

```dockerfile
FROM node:16.14.2-alpine3.14

RUN apk update && apk add less vim curl unzip sudo

RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
RUN unzip awscliv2.zip
RUN sudo ./aws/install

WORKDIR /workdir
```

Dockerfile内、以下の箇所でAWS CLIをインストールしています。AWSのサイトを参考にしています。

```dockerfile
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
RUN unzip awscliv2.zip
RUN sudo ./aws/install
```

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html

# イメージ作成

```sh
docker build --platform linux/amd64 -t aws .
```

プラットフォームが`arm`ではないイメージを作成するため、`--platform linux/amd64`を指定しています。

# コンテナ起動

```sh
docker run -it -v ~/.aws:/root/.aws -v "$PWD":/workdir --publish 3000:3000 --name aws --platform linux/amd64 aws bash
```

自分がコンテナを起動するときには、すでにローカル環境にAWS CLIの設定を行っていたため、`-v ~/.aws:/root/.aws`を記載し設定を同期しています。
こちらは以下のページを参考にしています。

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-docker.html

# Serverlessのインストール

コンテナ内で以下を実行します。

```sh
npm install -g serverless
```

Serverlessのインストールが完了したら、以下コマンドでプロジェクトを立ち上げることができます。

```sh
# sls でも可
serverless
```

# AWS CLI設定

AWS CLIのセットアップをする場合には以下コマンドを実行します。

```sh
aws configure
```

設定の方法は以下のページに記載されています。

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-quickstart.html