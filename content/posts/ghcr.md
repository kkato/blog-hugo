---
title: "GitHub Container Registry (ghcr.io) の使い方"
date: 2026-02-28T12:00:00+09:00
draft: false
tags: ["docker", "github", "github-actions", "container"]
---

GitHub Container Registry (ghcr.io) は、GitHub が提供するコンテナイメージのレジストリです。
Public Packages であれば無料で利用でき、GitHub Actions と組み合わせることでデータ転送料金も無料になります。

ghcr.io は2通りの使い方があります。
1. ローカルから Docker CLI を使ってイメージをビルドしてプッシュする方法
2. GitHub Actions を使ってコードの変更に合わせて自動でイメージをビルドしてプッシュする方法

今回はGitHub Actions を使って自動でイメージをビルドしてプッシュする方法を紹介します。

### GitHub Actions から自動でプッシュする

GitHub Actions を使えば、コードをプッシュするだけで自動的にイメージのビルドとプッシュができます。以下はワークフローの例です。

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

### トラブルシューティング

GitHub Actions から ghcr.io にプッシュする際に、以下のようなエラーが出ました。

```text
#16 ERROR: denied: permission_denied: write_package
------
 > pushing ghcr.io/kkato/test-app:latest with docker:
------
ERROR: failed to build: denied: permission_denied: write_package
```

ghcr.io 上のパッケージに対して、そのリポジトリの Actions がアクセスできるよう許可されていないことが原因でした。

まず GitHub プロフィールの「Packages」タブから対象のパッケージを探します。

![GitHubプロフィールのPackagesタブ](/images/ghcr/packages1.png)

パッケージのページに移動し、右側の「Package settings」をクリックします。

![パッケージページのPackage settings](/images/ghcr/packages2.png)

「Manage Actions access」セクションで「Add Repository」からリポジトリを追加し、Role を「Admin」に設定します。

![Manage Actions accessの設定](/images/ghcr/packages3.png)

再度 Actions を実行すれば、エラーが解消されました。

### 参考

- [GitHub Packages のドキュメント](https://docs.github.com/ja/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GGitHub Action で GHCR へのコンテナー イメージの Push が成功しない](https://blog.yukirii.dev/github-action-ghcr-push-error/)
