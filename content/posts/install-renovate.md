---
title: "Renovateの導入方法"
date: 2025-05-06T15:27:19+09:00
draft: false
---

手動での依存パッケージのアップデートが大変だったので、[Renovate](https://docs.renovatebot.com/)を導入してみました。
その手順と設定について簡単にメモします。

### Renovateとは？

Renovateは、依存関係の更新を自動化してくれるOSSのツールです。元々は個人開発から始まり、現在はイスラエルのセキュリティ企業 Mend社によって開発されています。

リポジトリ内の依存関係をスキャンし、新しいバージョンが利用可能な場合には自動でPull Request（PR）を作成してくれます。
Node.jsのnpmやyarnはもちろん、Go modules、Docker、Terraformなど多くの言語やパッケージマネージャーに対応しているのも特徴です。

対応しているパッケージマネージャーの一覧は[こちら](https://docs.renovatebot.com/modules/manager/#supported-managers)

### 導入手順

renovateはセルフホストする方法とGitHub Appを使う方法の2通りあります。手軽に使いたいので、今回はGitHub Appを使って導入します。

1. https://github.com/apps/renovate から「Configure」を押す

![](/images/install-renovate/configure.png)

2. 全てのリポジトリを選択し、導入する

全てのリポジトリを選択したとしても、renovateの設定ファイルがあるリポジトリのみrenovateが有効になります。
そのため、renovateの設定ファイルがないリポジトリではrenovateが無効化されます。

![](/images/install-renovate/repository-access.png)

3. renovateの設定ファイル`renovate.json`を作成し、リポジトリのrootにpushする

各リポジトリで`Configure Renovate`というPull Requestが自動的に作成されます。

![](/images/install-renovate/configure-renovate.png)

自動生成された`renovate.json`
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ]
}
```

#### `$schema`

Renovateにおける`$schema`は、`renovate.json`で使える設定項目やその値の種類・ルールを定義しています。
これにより、どの設定が有効かをエディタが理解できたり、エディタによる補完などが有効になったりします。

#### `extends`

`extends`では、Renovateがあらかじめ用意しているルールセットを指定することができます。
上記の例では`config:recommended`を指定しています。これには、どの言語やプロジェクトでも使える推奨設定が含まれています。
（以前は`config:recommended`ではなく、`config:base`と呼ばれていたみたいです。）

config:recommended の詳細については[こちら](https://docs.renovatebot.com/presets-config/#configrecommended) \
またその他設定項目については[こちら](https://docs.renovatebot.com/configuration-options/)

### まとめ

現在業務でRenovate を使ってみて感じたのは、カスタマイズ性が非常に高い反面、設定項目が多くてややとっつきにくいという点でした。しかし、プライベートの時間を使って軽く触ってみたところ、導入自体は驚くほど簡単で、いい意味で期待を裏切られました。最小限の設定でもすぐに使い始めることができましたが、もう少しドキュメントを読んで複雑な設定も試してみたいと思いました。
