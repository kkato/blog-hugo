---
title: "Argo CDをインストールしてGitOpsを始める"
date: 2026-02-20T11:00:00+09:00
draft: false
tags: ["kubernetes", "argocd", "helm"]
---

Argo CDはKubernetes向けのGitOpsツールで、Gitリポジトリに置いたマニフェストをクラスタに自動で反映してくれます。
今回はHelmを使ってArgo CDをインストールし、リポジトリの接続からApplicationの作成までをまとめました。

### 前提

- Kubernetesクラスタが構築済みであること
- Helmがインストール済みであること

### Argo CDのインストール

Helm Chartリポジトリを追加します。

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

`argocd` namespaceにインストールします。

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.8.13
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

### Gitリポジトリを接続する

Argo CDがGitリポジトリからマニフェストを取得するために、リポジトリの接続設定が必要です。

#### SSH鍵の作成

Argo CD専用のSSH鍵を作成します。

```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd -C "argocd" -N ""
```

#### GitHubにDeploy Keyを登録

生成した公開鍵をリポジトリのDeploy Keyとして登録します。

```bash
gh repo deploy-key add ~/.ssh/argocd.pub --repo <user>/<repo> --title "argocd"
```

#### Argo CDにSSH鍵を登録

秘密鍵をKubernetes SecretとしてArgo CDに登録します。

```bash
kubectl create secret generic repo-k8s -n argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:<user>/<repo> \
  --from-file=sshPrivateKey=~/.ssh/argocd
kubectl label secret repo-k8s -n argocd argocd.argoproj.io/secret-type=repository
```

### Applicationの作成

Argo CDではApplicationというカスタムリソースで、どのリポジトリのどのパスをどのnamespaceに同期するかを定義します。

#### 素のマニフェストを管理するApplication

Gitリポジトリに置いたKubernetesマニフェストをそのまま同期する場合の例です。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: recipe-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<user>/<repo>
    targetRevision: main
    path: apps/recipe-api
  destination:
    server: https://kubernetes.default.svc
    namespace: recipe-api
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

主要なフィールドの説明:

- `source.path` - リポジトリ内のマニフェストが置かれたディレクトリ
- `destination.namespace` - 同期先のnamespace
- `syncPolicy.automated` - Gitの変更を自動で検知・同期する
- `prune: true` - Gitから削除されたリソースをクラスタからも削除する
- `selfHeal: true` - クラスタ上で手動変更されてもGitの状態に戻す

#### Helm Chartを管理するApplication

Helm Chartリポジトリから直接インストールする場合の例です。values.yamlはGitリポジトリから参照できます。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argo-cd
      targetRevision: 7.8.13
      helm:
        valueFiles:
          - $values/addons/argocd/values.yaml
    - repoURL: git@github.com:<user>/<repo>
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

この例ではArgo CDが自分自身を管理しています（self-manage）。`sources` で複数のソースを指定し、Helm Chartのvalues.yamlをGitリポジトリから参照する構成です。`$values` は2つ目のsourceの `ref: values` に対応しています。

#### Applicationの適用

作成したApplicationをクラスタに適用します。

```bash
kubectl apply -f argocd/apps/recipe-api.yaml
kubectl apply -f argocd/addons/argocd.yaml
```

適用後はArgo CDが自動でGitリポジトリを監視し、変更があれば同期してくれます。

### ディレクトリ構成の例

Argo CDでGitOps運用する場合、以下のようなリポジトリ構成が管理しやすいです。

```text
k8s/
├── apps/              # 自作アプリ (素のK8sマニフェスト)
│   └── recipe-api/
│       └── deployment.yaml
├── addons/            # クラスタaddon (Helm values)
│   └── argocd/
│       └── values.yaml
└── argocd/            # Argo CD Application定義
    ├── apps/
    │   └── recipe-api.yaml
    └── addons/
        └── argocd.yaml
```

- `apps/` - マニフェストを変更してpushするだけで自動デプロイ
- `addons/` - values.yamlを変更してpushするだけでaddonの設定変更が反映
- `argocd/` - 新しいアプリやaddonを追加するときにApplication定義を追加
