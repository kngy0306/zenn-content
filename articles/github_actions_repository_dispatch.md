---
title: "GitHub Actionsで別リポジトリ(private)のファイル情報を取得する"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions"]
published: false
---

# 概要

- フロントエンド、バックエンドそれぞれ別のプライベートリポジトリで開発を行っている
- APIはGraphQLを使用している
- フロントエンドでは、バックエンドから出力された`schema.graphql`ファイルを参照して型ファイルを生成する

以上のようなアプリケーション構造を検討する場面があり、フロントエンドのリポジトリ（以下「FEリポジトリ」）から、バックエンドのリポジトリ（以下「BEリポジトリ」）内にある`schema.graphql`ファイルを参照する方法が課題としてありました。
その課題に対し、GitHub Actionsを用いてリポジトリ間でスキーマファイルを共有する方法を試しました。
同じような課題を抱えている方向けに、1つの参考になればと思いこの記事を書きました。

# 全体の流れ

1. BEリポジトリの任意ブランチで`graphql.schema`の変更を検知したらFEリポジトリへ通知を送信
2. 通知を受け取ったFEブランチは、GitHub ActionsでBEリポジトリの`graphql.schema`を参照
3. 参照した`graphql.schema`をFEブランチのファイルとして作成し、PRを作成

# 各リポジトリに配置するymlファイル

※`< >`で囲まれた文字は手元の状況によって変わるものを表しています

## BEリポジトリのymlファイル

BEリポジトリでは、指定したブランチで`graphql.schema`に変更があった場合にFEリポジトリに通知を送信します。

```yml
name: graphql.schemaファイルの更新をFEリポジトリに通知

on:
  push:
    branches:
      - "feature/hoge" # 任意のブランチ名

jobs:
  send_repository_dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: ファイル変更を検知
        uses: technote-space/get-diff-action@v6
        with:
          PATTERNS: schema.graphql # ①
      - name: FEリポジトリにイベントを通知する
        uses: peter-evans/repository-dispatch@v2
        with:
          repository: <owner>/<fe_repository_name>
          token: ${{ secrets.PAT }}
          event-type: graphql-schema-updated # ②
        if: env.GIT_DIFF
```

### ①

`PATTERNS:`に、変更を検知したいファイル名を記載しています。

### ②

`event-type`は任意の名前をつけることができますが、この後のFEリポジトリにて同じ名前を指定する必要があります。

## FEリポジトリのymlファイル

BEリポジトリから通知を受け取り、`graphql.schema`更新のPRを作成するymlファイル

```yml
name: BEリポジトリからの通知を受け取りPRを作成

on:
  repository_dispatch:
    types: [ graphql-schema-updated ] # ③

jobs:
  receive_repository_dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v2
      - name: generate github token
        id: generate_token
        uses: tibdex/github-app-token@v1
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
      - name: create PR
        run: |
          echo "=== PRのためのブランチを作成 ==="
          git config user.name "<user_name>" # ④
          git config user.email "<user_email>"
          git fetch --prune
          git switch -c "<任意のPR元ブランチ名>" "<origin/任意のPR元ブランチ名>"
          git merge --allow-unrelated-histories --squash "<FEリポジトリの任意ブランチ名>"
          echo "=== BEリポジトリのschema.graphqlファイルの内容をschema.graphqlファイルに保存 ===="
          curl -H "Accept: application/vnd.github.raw" \ # ⑤
               -H "Authorization: token ${GITHUB_TOKEN}" \
               -o "schema.graphql" \
               "https://raw.githubusercontent.com/<owner>/<repo>/<branch>/schema.graphql"
          echo "=== 差分をコミット ==="
          git add schema.graphql # ⑥
          git commit -m "BEリポジトリのschema.graphqlファイルが更新されました"
          git push origin "PR元のブランチ"
          curl -X POST \
               -H "Accept: application/vnd.github.raw" \
               -H "Authorization: token ${GITHUB_TOKEN}" \
               https://api.github.com/repos/<owner>/<repo>/pulls \
               -d '{ "title":"[bot]PR from github actions", "base":"<FEリポジトリの任意ブランチ名>", "head":"<任意のPR元ブランチ名>" }'
        env:
          GITHUB_TOKEN: ${{ steps.generate_token.outputs.token }}
```

### ③

②の`event-type`で指定した名前と同じ名前にします。

### ④

gitコマンドを用いてPR作成に必要なブランチ作成などを行っています。

### ⑤

curlコマンドを用いてBEリポジトリのファイルを参照し、FEリポジトリに`schema.graphql`ファイルに出力しています。

### ⑥

[GitHub REST API](https://docs.github.com/ja/rest/guides/getting-started-with-the-rest-api?apiVersion=2022-11-28&tool=curl)を用いてPRを作成しています。

## 使用しているActions

- [repository_dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [get-diff-action](https://github.com/technote-space/get-diff-action)
- [checkout](https://github.com/actions/checkout)

## 権限について

GitHub Actions内にて、GitHubAppsトークンを使用しています。他リポジトリを参照する際などに必要になります。以下のサイトを参考に作成しました。

https://zenn.dev/tmknom/articles/github-apps-token

# その他参考サイト

https://kaigi-on-rails-2021-mf-maintain-graphql-types-over-repositories.netlify.app/
