---
title: "Kubernetesをv1.30からv1.31にアップグレードした"
date: 2026-01-24T20:11:09+09:00
draft: false
tags: ["kubernetes"]
---

自宅で運用しているKubernetesクラスターをv1.30からv1.31にアップグレードしました。
kubeadmを使った標準的なアップグレード手順ですが、忘れがちなので備忘録として残しておきます。

公式ドキュメント: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

## コントロールプレーンノードのアップグレード

コントロールプレーンノードから順番にアップグレードしていきます。

### パッケージリポジトリの変更

まず、yumリポジトリの設定ファイルをv1.31用に変更します。

変更前（v1.30）:
```bash
[kkato@nuc01 ~]$ cat /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

変更後（v1.31）:
```bash
$ cat /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

参考: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/

### kubeadmのアップグレード

まずkubeadmパッケージをアップグレードします。

```bash
sudo yum upgrade kubeadm --setopt=disable_excludes=kubernetes
```

アップグレード可能なバージョンを確認します。

```bash
sudo kubeadm upgrade plan

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     nuc01     v1.30.1   v1.31.14
kubelet     nuc02     v1.30.1   v1.31.14
kubelet     nuc03     v1.30.1   v1.31.14
kubelet     nuc04     v1.30.1   v1.31.14

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            nuc01     v1.30.4    v1.31.14
kube-controller-manager   nuc01     v1.30.4    v1.31.14
kube-scheduler            nuc01     v1.30.4    v1.31.14
kube-proxy                          1.30.4     v1.31.14
CoreDNS                             v1.11.1    v1.11.3
etcd                      nuc01     3.5.12-0   3.5.24-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.31.14
```

アップグレードを実行します。

```bash
$ sudo kubeadm upgrade apply v1.31.14

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.31.14". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

### kubeletとkubectlのアップグレード

コントロールプレーンのkubeletとkubectlをアップグレードします。

```bash
sudo yum upgrade kubelet kubectl --setopt=disable_excludes=kubernetes
```

kubeletを再起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### ノードの確認

コントロールプレーンノードがv1.31.14にアップグレードされたことを確認します。

```console
NAME    STATUS   ROLES           AGE    VERSION
nuc01   Ready    control-plane   518d   v1.31.14
nuc02   Ready    <none>          518d   v1.30.1
nuc03   Ready    <none>          518d   v1.30.1
nuc04   Ready    <none>          518d   v1.30.1
```

## ワーカーノードのアップグレード

コントロールプレーンのアップグレードが完了したら、ワーカーノードを順番にアップグレードしていきます。

### パッケージリポジトリの変更とkubeadm upgrade node

コントロールプレーンと同様に、まずリポジトリをv1.31に変更してから、kubeadmをアップグレードします。
その後、ワーカーノードの設定を更新します。

```bash
$ sudo kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config634698628/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

### kubeletとkubectlのアップグレード

ノードをメンテナンスモードにして、Podを退避させます。

```bash
kubectl drain nuc02 --ignore-daemonsets --delete-emptydir-data
``

kubeletとkubectlをアップグレードします。

```bash
sudo yum upgrade kubelet kubectl --setopt=disable_excludes=kubernetes
```

kubeletを再起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

メンテナンスモードを解除して、ノードをスケジューリング対象に戻します。

```bash
kubectl uncordon nuc02
```

### ノードの確認

全てのワーカーノードに対して同じ手順を繰り返します。
全ノードのアップグレードが完了したことを確認します。

```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE    VERSION
nuc01   Ready    control-plane   518d   v1.31.14
nuc02   Ready    <none>          518d   v1.31.14
nuc03   Ready    <none>          518d   v1.31.14
nuc04   Ready    <none>          518d   v1.31.14
```

## kubectlクライアントのアップグレード

最後に、ローカルマシン（Mac）のkubectlもアップグレードします。

現在のバージョンを確認すると、クライアントがv1.30.1、サーバーがv1.31.14になっています。

```bash
$ kubectl version
Client Version: v1.30.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.31.14
```

macOS用（ARM64）のkubectlバイナリをダウンロードします。

```bash
curl -LO "https://dl.k8s.io/release/v1.31.14/bin/darwin/arm64/kubectl"
```

実行権限を付与して、適切な場所に配置します。

```bash
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
```

クライアントとサーバーのバージョンが揃ったことを確認します。

```bash
$ kubectl version
Client Version: v1.31.14
Kustomize Version: v5.4.2
Server Version: v1.31.14
```

これでKubernetesクラスターのアップグレードが完了しました。
