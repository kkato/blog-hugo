---
title: "tfactionでTerraformリポジトリをセットアップする"
date: 2026-02-25T12:00:00+09:00
draft: true
tags: ["terraform", "github-actions"]
---

[tfaction](https://github.com/suzuki-shunsuke/tfaction)はGitHub Actions上でTerraformのCI/CDを構築するためのアクション群です。
monolithリポジトリ構成で複数のworking directoryを管理でき、PRでplan・mergeでapplyという運用を実現できます。
今回はGCSバックエンドを使ったセットアップ手順をまとめました。

### 前提

- GCPプロジェクトが作成済みであること
- tfstateを保存するGCSバケットが作成済みであること
- GitHubリポジトリが作成済みであること

GCSバケットがない場合は以下で作成します。

```bash
gcloud storage buckets create gs://kkato-tfstate --project kkato-app --location asia-northeast1
```

### ディレクトリ構成

最終的なリポジトリ構成は以下のようになります。

```text
terraform/
├── .github/
│   └── workflows/
│       ├── test.yaml                    # PR時: terraform plan
│       ├── apply.yaml                   # main push時: terraform apply
│       ├── wc-plan.yaml                 # 再利用可能ワークフロー (plan)
│       ├── wc-apply.yaml                # 再利用可能ワークフロー (apply)
│       └── scaffold-working-dir.yaml    # working directory 作成用
├── aqua/
│   ├── aqua.yaml                        # ツールバージョン管理
│   └── imports/
│       └── tfaction.yaml                # tfaction CLI
├── k8s/                                 # working directory
│   ├── main.tf
│   ├── tfaction.yaml
│   └── aqua/
│       └── aqua.yaml
├── templates/
│   └── default/                         # scaffold テンプレート
│       └── main.tf
├── tfaction-root.yaml                   # tfaction グローバル設定
├── aqua-checksums.json
└── .gitignore
```

### tfaction-root.yaml

tfactionのグローバル設定ファイルです。`target_groups`でworking directoryのグループを定義します。

```yaml
---
plan_workflow_name: test
target_groups:
  - working_directory: k8s/
    target: k8s/
    template_dir: templates/default
tflint:
  enabled: true
aqua:
  update_checksum:
    enabled: true
    prune: true
```

`plan_workflow_name`はPR時に実行されるワークフロー名で、`test.yaml`の`name`と一致させます。`template_dir`はscaffoldワークフローで新しいworking directoryを作成するときに使われるテンプレートです。

### aquaによるツール管理

[aqua](https://aquaproj.github.io/)でTerraformやtfaction関連のCLIツールを管理します。

```yaml
# aqua/aqua.yaml
---
registries:
  - type: standard
    ref: v4.311.0 # renovate: depName=aquaproj/aqua-registry
packages:
  - name: hashicorp/terraform@v1.11.4
  - name: terraform-linters/tflint@v0.55.1
  - name: suzuki-shunsuke/tfcmt@v4.14.0
  - name: suzuki-shunsuke/github-comment@v6.3.2
imports:
  - imports/tfaction.yaml
```

```yaml
# aqua/imports/tfaction.yaml
---
packages:
  - name: suzuki-shunsuke/tfaction-go@v1.2.0
```

registryのrefに`# renovate:`コメントを付けておくと、Renovateでバージョンを自動更新できます。

### Working Directoryの作成

tfactionではworking directoryごとにTerraformの設定を持ちます。最初の`k8s/`ディレクトリを作成します。

```hcl
# k8s/main.tf
terraform {
  backend "gcs" {
    bucket = "kkato-tfstate"
    prefix = "k8s"
  }
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

provider "google" {
  project = "kkato-app"
}
```

working directoryごとのtfaction設定ファイルも置きます。特別な設定がなければ空で構いません。

```yaml
# k8s/tfaction.yaml
---
{}
```

working directory固有のツールが必要な場合は`k8s/aqua/aqua.yaml`に追加できます。

```yaml
# k8s/aqua/aqua.yaml
---
registries:
  - type: standard
    ref: v4.311.0 # renovate: depName=aquaproj/aqua-registry
packages: []
```

### Scaffoldテンプレート

新しいworking directoryを追加するときに使うテンプレートです。

```hcl
# templates/default/main.tf
terraform {
  backend "gcs" {
    bucket = "kkato-tfstate"
    prefix = "{{.Target}}"
  }
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}

provider "google" {
  project = "kkato-app"
}
```

`{{.Target}}`はscaffold実行時にworking directory名に置き換えられます。

### GitHub Actionsワークフロー

tfactionのワークフローは、トリガー用のワークフローと再利用可能ワークフローに分けて構成します。

#### test.yaml（PR時のplan）

PRが作成されると`list-targets`で変更のあるworking directoryを検出し、matrixでplanを実行します。

```yaml
# .github/workflows/test.yaml
---
name: test
on: pull_request_target
concurrency:
  group: ${{ github.workflow }}--${{ github.head_ref }}
  cancel-in-progress: true
permissions: {}
jobs:
  setup:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      targets: ${{ steps.list-targets.outputs.targets }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          persist-credentials: false
      - uses: aquaproj/aqua-installer@e2d0136abcf70b7a2f93480c8c7bfed50a348d5c # v4.0.0
        with:
          aqua_version: v2.46.0
      - uses: suzuki-shunsuke/tfaction/list-targets@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        id: list-targets

  plan:
    uses: ./.github/workflows/wc-plan.yaml
    needs: setup
    if: join(fromJSON(needs.setup.outputs.targets), '') != ''
    with:
      targets: ${{ needs.setup.outputs.targets }}
      ref: ${{ github.event.pull_request.head.sha }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
```

`pull_request_target`を使うことで、forkからのPRでもSecretsにアクセスできます。`list-targets`はGitの差分からどのworking directoryが変更されたかを検出します。

#### apply.yaml（main pushでのapply）

mainブランチへのpush時に`terraform apply`を実行します。

```yaml
# .github/workflows/apply.yaml
---
name: apply
on:
  push:
    branches: [main]
permissions: {}
env:
  TFACTION_IS_APPLY: "true"
jobs:
  setup:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    outputs:
      targets: ${{ steps.list-targets.outputs.targets }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false
      - uses: aquaproj/aqua-installer@e2d0136abcf70b7a2f93480c8c7bfed50a348d5c # v4.0.0
        with:
          aqua_version: v2.46.0
      - uses: suzuki-shunsuke/tfaction/list-targets@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        id: list-targets

  apply:
    uses: ./.github/workflows/wc-apply.yaml
    needs: setup
    if: join(fromJSON(needs.setup.outputs.targets), '') != ''
    with:
      targets: ${{ needs.setup.outputs.targets }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
```

`TFACTION_IS_APPLY: "true"`を設定することで、tfactionがapplyモードで動作します。

#### wc-plan.yaml（再利用可能ワークフロー）

```yaml
# .github/workflows/wc-plan.yaml
---
name: plan
on:
  workflow_call:
    inputs:
      targets:
        required: true
        type: string
      ref:
        required: true
        type: string
permissions: {}
jobs:
  plan:
    timeout-minutes: 30
    name: "plan (${{ matrix.target.target }})"
    runs-on: ${{ matrix.target.runs_on }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    env:
      TFACTION_TARGET: ${{ matrix.target.target }}
      TFACTION_WORKING_DIR: ${{ matrix.target.working_directory }}
      TFACTION_JOB_TYPE: ${{ matrix.target.job_type }}
      GH_COMMENT_SHA1: ${{ inputs.ref }}
      TFCMT_SHA: ${{ inputs.ref }}
    strategy:
      fail-fast: true
      matrix:
        target: ${{ fromJSON(inputs.targets) }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          ref: ${{ inputs.ref }}
          persist-credentials: false
      - uses: aquaproj/aqua-installer@e2d0136abcf70b7a2f93480c8c7bfed50a348d5c # v4.0.0
        with:
          aqua_version: v2.46.0
      - uses: suzuki-shunsuke/tfaction/setup@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
      - uses: suzuki-shunsuke/tfaction/test@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
      - uses: suzuki-shunsuke/tfaction/plan@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
```

`setup`でterraform initやproviderのインストールが行われ、`test`でtflintなどの静的解析、`plan`でterraform planが実行されます。結果はtfcmtによってPRにコメントされます。

#### wc-apply.yaml（再利用可能ワークフロー）

```yaml
# .github/workflows/wc-apply.yaml
---
name: apply
on:
  workflow_call:
    inputs:
      targets:
        required: true
        type: string
permissions: {}
env:
  TFACTION_IS_APPLY: "true"
jobs:
  apply:
    timeout-minutes: 30
    name: "apply (${{ matrix.target.target }})"
    runs-on: ${{ matrix.target.runs_on }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    env:
      TFACTION_TARGET: ${{ matrix.target.target }}
      TFACTION_WORKING_DIR: ${{ matrix.target.working_directory }}
      TFACTION_JOB_TYPE: ${{ matrix.target.job_type }}
    strategy:
      fail-fast: false
      matrix:
        target: ${{ fromJSON(inputs.targets) }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false
      - uses: aquaproj/aqua-installer@e2d0136abcf70b7a2f93480c8c7bfed50a348d5c # v4.0.0
        with:
          aqua_version: v2.46.0
      - uses: suzuki-shunsuke/tfaction/setup@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
      - uses: suzuki-shunsuke/tfaction/apply@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
      - uses: suzuki-shunsuke/tfaction/create-follow-up-pr@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        if: failure()
        with:
          github_token: ${{ github.token }}
```

applyが失敗した場合、`create-follow-up-pr`がフォローアップ用のPRを自動作成します。

#### scaffold-working-dir.yaml（新規working directory作成）

GitHub Actionsの`workflow_dispatch`から手動でworking directoryを作成できます。

```yaml
# .github/workflows/scaffold-working-dir.yaml
---
name: Scaffold a working directory
run-name: Scaffold a working directory (${{ inputs.working_dir }})
on:
  workflow_dispatch:
    inputs:
      working_dir:
        description: working directory
        required: true
permissions: {}
env:
  TFACTION_WORKING_DIR: ${{ github.event.inputs.working_dir }}
jobs:
  scaffold:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false
      - uses: aquaproj/aqua-installer@e2d0136abcf70b7a2f93480c8c7bfed50a348d5c # v4.0.0
        with:
          aqua_version: v2.46.0
      - uses: suzuki-shunsuke/tfaction/scaffold-working-dir@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
      - uses: suzuki-shunsuke/tfaction/create-scaffold-pr@4562d910bbacecf384c8aeda25332284fcf05f38 # v1.20.1
        with:
          github_token: ${{ github.token }}
```

GitHubのActionsタブから「Run workflow」でworking directory名を入力して実行すると、テンプレートからファイルが生成されPRが作成されます。

### Working Directoryの追加方法

新しいworking directoryを追加するときは以下の手順です。

1. `tfaction-root.yaml`の`target_groups`にエントリを追加する
2. scaffoldワークフローを実行するか、手動でディレクトリを作成する
3. PRを作成してplanを確認し、mergeしてapplyする

### 動作確認

ローカルでterraform initが通ることを確認します。

```bash
terraform -chdir=k8s init
terraform -chdir=k8s validate
```

GCSバックエンドへの認証が必要な場合は、事前にApplication Default Credentialsを設定します。

```bash
gcloud auth application-default login
```
