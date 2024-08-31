---
title: "Rocky Linuxにcontainerdをインストールする方法"
date: 2024-06-30T16:46:03+09:00
draft: false
---

以前Kubernetesクラスタを構築するときに、Rocky Linuxでcontainerdをインストールする方法についてあまり情報がなかったので、とても苦労しました。
なので今回はその時に調べた内容を記事にしてみました。

次のコマンドでcontainerdをインストールできます。

```sh
# yumにCentOS用のdockerリポジトリを追加する
sudo yum config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# docker-ceリポジトリに含まれているcontainerd.ioのパッケージをインストールする
sudo yum install -y containerd.io

# containerdのデフォルト設定ファイルを生成する
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```


そして最後にcontainerdを有効化します。

```sh
systemctl enable --now containerd.service
```

### 参考
- [Using containerd without docker](https://medium.com/@DannielWhatever/using-containerd-without-docker-9d08332781b4) (Installing containerdの部分)
