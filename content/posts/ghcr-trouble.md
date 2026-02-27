---
title: "GitHub Container Registry (ghcr.io) の使い方"
date: 2026-02-26T12:00:00+09:00
draft: true
tags: ["docker", "github", "github-actions", "container"]
---

GitHub Container Registry (ghcr.io) は、GitHub が提供するコンテナイメージのレジストリです。Docker Hub の代わりに GitHub 上でイメージを管理でき、GitHub Actions との連携もスムーズです。
個人リポジトリで Docker イメージを管理する際に使ったので、手順をまとめました。

### ローカルからの操作

#### ログイン

ghcr.io へのログインには GitHub の Personal Access Token (PAT) が必要です。PAT には `write:packages` スコープを付与しておきます。

```bash
# PAT を使って ghcr.io にログイン
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
```

#### イメージのビルドとプッシュ

```bash
# イメージをビルド
docker build -t ghcr.io/kkato/recipe-api:latest .

# ghcr.io にプッシュ
docker push ghcr.io/kkato/recipe-api:latest
```

タグの形式は `ghcr.io/<ユーザー名>/<イメージ名>:<タグ>` です。

#### イメージのプル

```bash
docker pull ghcr.io/kkato/recipe-api:latest
```

パブリックなイメージであればログイン不要でプルできます。

### GitHub Actions から自動でプッシュする

GitHub Actions を使えば、コードをプッシュするだけで自動的にイメージのビルドとプッシュができます。以下は [kkato/recipe-api](https://github.com/kkato/recipe-api) で使っているワークフローの例です。

```yaml
name: Docker Build and Push

on:
  push:
    branches:
      - main

# GHCR へのプッシュには packages: write が必要
permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      # ソースコードをチェックアウト
      - name: Check out code
        uses: actions/checkout@v4

      # ghcr.io にログイン
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          # デフォルトの GITHUB_TOKEN を利用
          password: ${{ secrets.GITHUB_TOKEN }}

      # Docker イメージをビルドしてプッシュ
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

#### ポイント

- `permissions` で `packages: write` を指定することで、`GITHUB_TOKEN` に ghcr.io への書き込み権限が付与される
- `docker/login-action` で ghcr.io にログインし、`docker/build-push-action` でビルドとプッシュをまとめて行う
- PAT を別途用意する必要はなく、`secrets.GITHUB_TOKEN` がそのまま使える

### イメージの公開設定

ghcr.io にプッシュしたイメージはデフォルトではプライベートです。公開したい場合は、GitHub の **Packages** ページからパッケージの設定を開き、Visibility を Public に変更します。

### 参考

- [GitHub Packages のドキュメント](https://docs.github.com/ja/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
