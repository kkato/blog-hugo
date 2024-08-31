---
title: "TerraformのbackendとしてGCSを使う"
date: 2024-08-31T20:56:16+09:00
draft: false
categories: ["terraform"]
---

参考: [Backend Type: gcs](https://developer.hashicorp.com/terraform/language/settings/backends/gcs)

## サービスアカウントの作成

まず初めにサービスアカウントを作成します。

今回はコンソールから`terraformer`というサービスアカウントを作成し、`Storage オブジェクト管理者`の権限を付与します。

![](/images/terraform-gcs/storage-object-admin.png)

「キー」のタブから「鍵を追加」を選択し、JSONキーを作成します。

## GCSバケットの作成

コンソールから`kkato-terraform-state`というGCSのバケットを作成します。


## Terraform側の設定

`main.tf`というファイルを作成し、backendとproviderの設定をします。

```
terraform {
  backend "gcs" {
    bucket = "kkato-terraform-state"
    prefix = "kkato-app"
  }
}

provider "google" {
  project = "kkato-app"
  region  = "asia-northeast1"
}
```

その後`terraform init`を実行すると、`tfstate`をGCSで管理することができます。
(今回はすでにtfstateがローカルにある状態で、`terraform.tfstate`をGCSにコピーしています。)

```
% terraform init

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "gcs" backend. No existing state was found in the newly
  configured "gcs" backend. Do you want to copy this state to the new "gcs"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes


Successfully configured the backend "gcs"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/local from the dependency lock file
- Finding latest version of hashicorp/google...
- Using previously-installed hashicorp/local v2.5.1
- Installing hashicorp/google v6.0.1...
- Installed hashicorp/google v6.0.1 (signed by HashiCorp)

Terraform has made some changes to the provider dependency selections recorded
in the .terraform.lock.hcl file. Review those changes and commit them to your
version control system if they represent changes you intended to make.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```
