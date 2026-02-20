---
title: "Helmfileの使い方"
date: 2026-02-20T12:00:00+09:00
draft: false
tags: ["kubernetes", "helm", "helmfile"]
---

HelmfileはHelmの操作を宣言的に管理するためのツールです。
複数のHelm Chartをまとめて管理したいときや、インストール内容をGitで管理したいときに便利です。

### Helmとの違い

Helmだけの場合、インストール時にコマンドで毎回オプションを指定する必要があります。

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.8.13 \
  -f values.yaml
```

Helmfileを使うと、これをYAMLファイルに宣言的に書けます。

```yaml
releases:
  - name: argocd
    namespace: argocd
    createNamespace: true
    chart: argo/argo-cd
    version: 7.8.13
    values:
      - values.yaml
```

何をどのバージョンでインストールしたかがファイルに残るので、Gitで管理できます。

### インストール

macOSの場合は以下のコマンドでインストールできます。

```bash
brew install helmfile
```

helm-diffプラグインも必要です。

```bash
helm plugin install https://github.com/databus23/helm-diff
```

### helmfile.yamlの書き方

基本的な構成は `repositories` と `releases` の2つです。

```yaml
repositories:
  - name: argo
    url: https://argoproj.github.io/argo-helm
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

releases:
  - name: argocd
    namespace: argocd
    createNamespace: true
    chart: argo/argo-cd
    version: 7.8.13
    values:
      - argocd/values.yaml

  - name: ingress-nginx
    namespace: ingress-nginx
    createNamespace: true
    chart: ingress-nginx/ingress-nginx
    version: 4.12.0
    values:
      - ingress-nginx/values.yaml
```

`repositories` でHelmリポジトリを定義し、`releases` で各Chartのインストール設定を記述します。
複数のaddonをまとめて管理できるのがHelmfileの強みです。

### 基本コマンド

#### インストール・更新

定義した内容をクラスタに適用します。

```bash
helmfile sync
```

未インストールのリリースはインストール、既存のリリースはアップグレードされます。

#### 差分確認

適用前に変更内容を確認できます。

```bash
helmfile diff
```

#### 状態確認

```bash
helmfile status
```

#### 削除

定義したリリースをすべて削除します。

```bash
helmfile destroy
```

#### テンプレート確認

生成されるマニフェストを確認できます。実際には適用されません。

```bash
helmfile template
```

### ディレクトリ構成の例

addonごとにvalues.yamlを分けると管理しやすいです。

```text
addons/
├── helmfile.yaml
├── argocd/
│   └── values.yaml
└── ingress-nginx/
    └── values.yaml
```
