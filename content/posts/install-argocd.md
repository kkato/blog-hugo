---
title: "HelmfileでArgo CDをインストールする"
date: 2026-02-20T11:00:00+09:00
draft: false
tags: ["kubernetes", "argocd", "helmfile"]
---

Argo CDをKubernetesクラスタにインストールする方法はいくつかありますが、今回はHelmfileを使って管理する方法をまとめました。
Helmfileを使うことで、Argo CDのバージョンやカスタム設定をGitで宣言的に管理できます。

### 前提

- Kubernetesクラスタが構築済みであること
- Helmがインストール済みであること

### Helmfileのインストール

macOSの場合は以下のコマンドでインストールできます。

```bash
brew install helmfile
```

Helmfileが依存するhelm-diffプラグインもインストールします。

```bash
helm plugin install https://github.com/databus23/helm-diff
```

### ディレクトリ構成

今回は以下のようなディレクトリ構成で管理します。

```
addons/
├── helmfile.yaml
└── argocd/
    └── values.yaml
```

### helmfile.yamlの作成

Argo CDのHelm Chartリポジトリとリリースを定義します。

```yaml
repositories:
  - name: argo
    url: https://argoproj.github.io/argo-helm

releases:
  - name: argocd
    namespace: argocd
    createNamespace: true
    chart: argo/argo-cd
    version: 7.8.13
    values:
      - argocd/values.yaml
```

### values.yamlの作成

カスタム設定を記述するためのファイルを用意します。デフォルト設定のまま使う場合は空でも問題ありません。

```yaml
## ArgoCD custom values
```

設定可能な値は以下のコマンドで確認できます。

```bash
helm show values argo/argo-cd
```

### Argo CDのインストール

helmfileを使ってArgo CDをインストールします。

```bash
cd addons
helmfile sync
```

Podが正常に起動していることを確認します。

```bash
kubectl get pods -n argocd
```

### Argo CD CLIのインストール

```bash
brew install argocd
```

### Argo CD UIにアクセスする

port-forwardでArgo CDのUIにアクセスできます。

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

ブラウザで `http://localhost:8080` にアクセスします。

初期パスワードは以下のコマンドで取得できます。ユーザー名は`admin`です。

```bash
argocd admin initial-password -n argocd
```

### Argo CDのアップグレード

`helmfile.yaml` の `version` を変更して再度 `helmfile sync` を実行するだけでアップグレードできます。

```bash
cd addons
helmfile sync
```

差分を事前に確認したい場合は `helmfile diff` が便利です。

```bash
helmfile diff
```
