---
title: "PostgreSQL Operatorの比較"
date: 2026-02-20T15:00:00+09:00
draft: false
tags: ["kubernetes", "postgresql"]
---

Kubernetes上でPostgreSQLを運用するためのOperatorが複数存在します。以前どれを選ぶか検討する機会があったので、主要な4つのオペレーターを比較してみました。

### 比較表

| | [PGO](https://github.com/CrunchyData/postgres-operator) | [CloudNativePG](https://github.com/cloudnative-pg/cloudnative-pg) | [Zalando](https://github.com/zalando/postgres-operator) | [StackGres](https://github.com/ongres/stackgres) |
|---|---|---|---|---|
| 開発元 | Crunchy Data | EDB / CNCF | Zalando | OnGres |
| GitHub Stars | 4,400 | 8,000 | 5,100 | 1,400 |
| ライセンス (Operator) | Apache 2.0 | Apache 2.0 | MIT | AGPL-3.0 |
| ライセンス (コンテナイメージ) | 独自 (制限あり) | Apache 2.0 | MIT | AGPL-3.0 |
| Patroni | 使用 | 不使用 (独自実装) | 使用 (Patroni開発元) | 使用 |
| CNCF | - | Sandbox | - | - |

### Star History

[![Star History Chart](https://api.star-history.com/svg?repos=CrunchyData/postgres-operator,cloudnative-pg/cloudnative-pg,zalando/postgres-operator,ongres/stackgres&type=Date)](https://star-history.com/#CrunchyData/postgres-operator&cloudnative-pg/cloudnative-pg&zalando/postgres-operator&ongres/stackgres&Date)

### 各オペレーターの特徴

#### PGO (Crunchy Data)

Crunchy Data社が開発するオペレーターで、Patroniを使ったHAに対応しています。Operator自体のソースコードはApache 2.0ですが、Crunchy Data が配布するコンテナイメージは独自のライセンス（[Crunchy Data Developer Program](https://www.crunchydata.com/developers/terms-of-use)）で提供されています。50名以上の組織では本番利用に有償サブスクリプションが必要になるため、注意が必要です。自前でイメージをビルドすれば回避できますが、手間がかかります。

#### CloudNativePG

EDBが開発を始め、現在はCNCF Sandboxプロジェクトになっています。Patroniに依存せず、独自のInstance Managerが各Pod内で動作し、Kubernetes APIを直接利用してリーダー選出やフェイルオーバーを行います。ライセンスもOperator・コンテナイメージともにApache 2.0で、制限なく利用できます。Star数も最も多く、コミュニティの勢いがあります。

#### Zalando postgres-operator

Zalando社が開発するオペレーターで、Patroniの開発元でもあります。MITライセンスで制限なく利用可能です。Spilo（PostgreSQL + Patroni をパッケージングしたDockerイメージ）をベースにしています。ただし、CloudNativePGと比較すると開発のペースは落ち着いてきている印象です。

#### StackGres

OnGres社が開発するオペレーターで、Patroni、PgBouncer、WAL-Gなどを含むフルスタック構成です。AGPLライセンスのためSaaSとして提供する場合はソースコード開示義務があり、組織のポリシーによっては導入が難しい場合があります。商用版のStackGres Enterpriseも提供されています。

### まとめ

ライセンスの自由度とコミュニティの活発さを重視するならCloudNativePGが現時点では最も有力な選択肢だと思います。Patroniに依存しない独自のHA実装もKubernetesネイティブなアプローチとして好印象です。
