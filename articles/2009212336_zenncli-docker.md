---
title: "Zenn CLI環境をdockerで構築し、Github経由で記事投稿を行う"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zenn", "docker", "GitHub", "nodejs", "bash"]
published: false
---

# はじめに

こんにちは。たかぱい（[@takapy0210](https://twitter.com/takapy0210)）です。

みなさんご存知の通り、ZennとGitHubリポジトリを連携することで、ローカルの好きなエディターで投稿コンテンツの作成・編集ができます。  
このローカルでの執筆時には「[Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli)」を導入すると、Markdownファイルの作成がスムーズになったり、コンテンツをプレビュー表示することが可能になります。  
このZenn CLIはNode.js製のため、はじめて使う場合は別途環境構築が必要になります。

ローカルに直接環境構築しても良いのですが、環境をあまり汚したくない人も多いのでは？と思い、このZenn CLI環境をdockerで構築してみました。

@[tweet](https://twitter.com/takapy0210/status/1308055724061126656)

コードは下記リポジトリで公開しています。  
https://github.com/takapy0210/zenn-content

また、本記事も以降で紹介するdocker環境で作成したものになります。

# コンテナ環境の準備

:::message
docker自体についての説明は巷にあふれているので、今回は割愛します。  
:::

Githubと連携するローカルディレクトリに`Dockerfile`と`docker-compose.yml`の2ファイルを用意します。  
また、記事のmdファイルを自動生成する際、ファイル名に`日付prefix`をデフォルトで付与するためのスクリプトも用意しました。

## Dockerfile
dockerfileです。

```docker
FROM node:12
LABEL takapy <hgoehoge@gmail.com>

WORKDIR /workspace

RUN apt-get -y update && apt-get install -y --no-install-recommends \
        git \
    && apt-get -y clean \
    && rm -rf /var/lib/apt/lists/*

RUN npm init --yes \
    && npm install -g zenn-cli@latest

# Locale Japanese
ENV LC_ALL=ja_JP.UTF-8
# Timezone jst
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

## docker-compose.yml
docker-composeファイルです。

```yml
version: '3.8'

x-template: &template
  build:
    context: .
  volumes:
    - ./:/workspace

services:
  zenn-init:
    container_name: zenn-init
    image: zenn-init:latest
    command: npx zenn init
    <<: *template
  zenn-preview:
    container_name: zenn-preview
    image: zenn-preview:latest
    ports:
      - "8000:8000"
    command: npx zenn preview
    <<: *template
  zenn-new-article:
    container_name: zenn-new-article
    image: zenn-new-article:latest
    command: /bin/bash new-article.sh
    <<: *template
```

## new-article
新規記事のテンプレートを作成するスクリプトです。  
初期ファイル名に日付prefixをデフォルトでつけたいがために作成しました。（ファイル名は`new-article.sh`）

```bash
#!/bin/bash

# get date
slug=$(date '+%y%m%d%H%M')

# create article
npx zenn new:article --slug "$slug"_hoge --title タイトル --type tech --emoji 🍀
```

# zenn CLI環境のビルド&初期化

コンテナをビルドし、zenn環境の初期化処理を行います。  
`docker-compose up -d zenn-init`を実行すると、カレントディレクトリに`articles`や`books`といったディレクトリが自動的に生成されます。

```bash
# ビルド
$ docker-compose build

# 初期化
$ docker-compose up -d zenn-init

>>>
Creating zenn-init ... done
Attaching to zenn-init
zenn-init       |
zenn-init       |   🎉Done!
zenn-init       |   早速コンテンツを作成しましょう
zenn-init       |
zenn-init       |   👇新しい記事を作成する
zenn-init       |   $ zenn new:article
zenn-init       |
zenn-init       |   👇新しい本を作成する
zenn-init       |   $ zenn new:book
zenn-init       |
zenn-init       |   👇表示をプレビューする
zenn-init       |   $ zenn preview
zenn-init       |
zenn-init exited with code 0
```

:::message
このビルドと初期化処理は初回のみ行えばOKです。  
定期的に記事を作成する場合に必要なのは↓以降の手順のみです。
:::

# 記事の作成
下記コマンドで新規記事（.mdファイル）を作成できます。  
時系列で管理しやすいように、初期ファイル名は`YYMMDDhhmm_hoge.md`となるようにしています。

```bash
# 新規記事のテンプレートファイルを作成
$ docker-compose up -d zenn-new-article
```

# 作成した記事のプレビューを表示
記事を作成したら、プレビューで意図した内容になっているか確認します。  
（ブラウザで`http://localhost:8000`にアクセスして確認できます）

```bash
# 記事のプレビューを表示
$ docker-compose up zenn-preview

>>>
👀 Preview on http://localhost:8000
```
プレビューで問題がなければ、Githubにpushして終わりです！

# 最後に
記事データをGitHubのリポジトリで管理できるのは結構便利だと思っているので、これを機に試してみてはいかがでしょうか。


# 参考
- [Githubとの連携方法](https://zenn.dev/zenn/articles/connect-to-github)
- [Zenn CLIの使い方](https://zenn.dev/zenn/articles/install-zenn-cli)
- [ZennのMarkdown記法](https://zenn.dev/zenn/articles/markdown-guide)
