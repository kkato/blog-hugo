---
title: "Helmコマンド まとめ"
date: 2025-12-27T09:37:48+09:00
draft: false
tags: ["kubernetes", "command"]
---

HelmはKubernetes向けのパッケージマネージャーで、複雑なアプリケーションのデプロイを簡単にしてくれるツールです。
Helmコマンドをよく忘れてしまうので、基本的なコマンドを備忘録としてまとめました。

## インストール方法

macOSの場合は以下のコマンドでインストールできます。
```bash
brew install helm
```

## リポジトリ管理

### リポジトリの追加
```bash
# リポジトリを追加する
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### リポジトリの一覧表示
```bash
# 追加されているリポジトリを確認する
helm repo list
```

### リポジトリの更新
```bash
# リポジトリのインデックスを更新する（新しいチャートバージョンを取得）
helm repo update
```

## チャートの検索・確認

### チャートの検索
```bash
# リポジトリからチャートを検索する
helm search repo nginx
helm search repo mysql
```

### チャート情報の表示
```bash
# チャートの詳細情報を表示する
helm show chart bitnami/nginx
helm show values bitnami/nginx
```

## リリース管理

### インストール
```bash
# チャートをインストールする
helm install my-nginx bitnami/nginx

# 名前空間を指定してインストール
helm install my-nginx bitnami/nginx -n production --create-namespace

# カスタム値でインストール
helm install my-nginx bitnami/nginx --set service.type=LoadBalancer
```

### リリース一覧の表示
```bash
# インストール済みのリリースを表示する
helm list

# 全ての名前空間のリリースを表示する
helm list -A

# 特定の名前空間のリリースを表示する
helm list -n production
```

### リリース情報の確認
```bash
# リリースの詳細情報を表示する
helm get all my-nginx

# リリースの設定値を表示する
helm get values my-nginx

# リリースの履歴を表示する
helm history my-nginx
```

### アップグレード
```bash
# リリースをアップグレードする
helm upgrade my-nginx bitnami/nginx

# 新しい値でアップグレード
helm upgrade my-nginx bitnami/nginx --set image.tag=1.21.0
```

### ロールバック
```bash
# 前のバージョンにロールバックする
helm rollback my-nginx

# 特定のリビジョンにロールバックする
helm rollback my-nginx 2
```

### アンインストール
```bash
# リリースを削除する
helm uninstall my-nginx

# 名前空間を指定して削除
helm uninstall my-nginx -n production
```

## 便利なオプション

### ドライラン
実際にはインストールせずに、何が実行されるかを確認できます。
```bash
helm install my-nginx bitnami/nginx --dry-run --debug
helm upgrade my-nginx bitnami/nginx --dry-run --debug
```

### テンプレートの確認
```bash
# 生成されるKubernetesマニフェストを確認する
helm template my-nginx bitnami/nginx
```
