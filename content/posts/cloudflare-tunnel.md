---
title: "Cloudflare Tunnelを使っておうちk8sクラスタにアクセスする"
date: 2026-03-01T13:27:12+09:00
draft: false
tags: ["kubernetes", "cloudflare", "argocd"]
---

Cloudflare Tunnelを使うと、パブリックIPやポート開放なしにKubernetesクラスタ上のサービスをインターネットに公開できます。
今回はCloudflare Tunnelの作成から、cloudflare-tunnel-ingress-controllerを使ってKubernetesのIngressリソースで自動的にトンネル経由の公開を行う設定までをまとめました。

### Cloudflare Tunnelの仕組み

通常、クラスタ内のサービスを外部に公開するにはパブリックIPの取得やルーターのポート開放が必要です。Cloudflare Tunnelを使うとクラスタ側からCloudflareへアウトバウンド接続を張るだけで、Cloudflare経由で外部からのアクセスを受けられるようになります。

cloudflare-tunnel-ingress-controllerを使うと、KubernetesのIngressリソースを作成するだけで自動的にCloudflare Tunnelのルーティングが設定されます。通常のIngress Controllerと同じ使い勝手でトンネル経由の公開が可能です。

### Cloudflare Tunnelの作成

Cloudflareダッシュボードからトンネルを作成します。

**ネットワーク > Tunnels** に移動し、「トンネルを作成」をクリックします。

![](/images/cloudflare-tunnel/create-cloudflare-tunnel1.png)

トンネル名を入力します。ここでは `kkato.app` としました。

![](/images/cloudflare-tunnel/create-cloudflare-tunnel2.png)

トンネルが作成されると、環境のセットアップ画面が表示されます。ここに表示されるトンネルトークンを後ほど使います。

![](/images/cloudflare-tunnel/create-cloudflare-tunnel3.png)

作成が完了するとトンネルの概要が表示されます。

![](/images/cloudflare-tunnel/create-cloudflare-tunnel4.png)

### Cloudflare APIトークンの作成

cloudflare-tunnel-ingress-controllerがCloudflare APIを操作するためのトークンを作成します。

[Cloudflareダッシュボード](https://dash.cloudflare.com/profile/api-tokens)のAPIトークンページにアクセスし、「Create Token」から「Create Custom Token」を選択します。

以下の権限を設定します。

- **Account** / **Cloudflare Tunnel** / **Edit**
- **Zone** / **DNS** / **Edit**

### シークレットをGCP Secret Managerに登録

External Secrets Operatorを使ってGCP Secret ManagerからKubernetesのSecretを自動生成する構成にしています。以下の3つのシークレットを登録します。

```bash
echo -n "<your-api-token>" | gcloud secrets create cloudflare-api-token --data-file=-
echo -n "<your-account-id>" | gcloud secrets create cloudflare-account-id --data-file=-
echo -n "<your-tunnel-name>" | gcloud secrets create cloudflare-tunnel-name --data-file=-
```

アカウントIDはCloudflareダッシュボードのURLやトンネルトークンのデコード結果から確認できます。

### ExternalSecretの作成

GCP Secret Managerから取得したシークレットをKubernetes Secretとして同期するExternalSecretを定義します。

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cloudflare-credentials
  namespace: cloudflare-tunnel-ingress-controller
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  target:
    name: cloudflare-credentials
  data:
    - secretKey: api_token
      remoteRef:
        key: cloudflare-api-token
    - secretKey: account_id
      remoteRef:
        key: cloudflare-account-id
    - secretKey: tunnel_name
      remoteRef:
        key: cloudflare-tunnel-name
```

### cloudflare-tunnel-ingress-controllerのインストール

Argo CD Applicationとして定義します。Helm Chartとvalues.yaml、そしてExternalSecretを含むGitリポジトリを組み合わせたマルチソース構成です。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudflare-tunnel-ingress-controller
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://helm.strrl.dev
      chart: cloudflare-tunnel-ingress-controller
      targetRevision: 0.0.22
      helm:
        valueFiles:
          - $values/addons/cloudflare-tunnel-ingress-controller/values.yaml
    - repoURL: git@github.com:kkato/k8s
      targetRevision: main
      ref: values
    - repoURL: git@github.com:kkato/k8s
      targetRevision: main
      path: addons/cloudflare-tunnel-ingress-controller
  destination:
    server: https://kubernetes.default.svc
    namespace: cloudflare-tunnel-ingress-controller
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

`ServerSideApply=true` を指定しています。これはCRDなど大きなリソースで `metadata.annotations` のサイズ上限（262144バイト）を超える問題を回避するためです。

values.yamlでは、トンネル名とシークレットの参照を設定します。

```yaml
cloudflare:
  tunnelName: kkato.app
  secretRef:
    name: cloudflare-credentials
    accountIDKey: account_id
    tunnelNameKey: tunnel_name
    apiTokenKey: api_token
```

### Ingressリソースでサービスを公開する

cloudflare-tunnel-ingress-controllerが動作していれば、`ingressClassName: cloudflare-tunnel` を指定したIngressリソースを作成するだけでサービスが公開されます。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: cloudflare-tunnel
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

Ingressを適用するとcloudflare-tunnel-ingress-controllerが自動的にCloudflare Tunnel側のルーティングを設定してくれます。DNSレコードもCloudflareに自動登録されるため、追加の設定は不要です。
