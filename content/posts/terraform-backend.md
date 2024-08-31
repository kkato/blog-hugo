---
title: "TerraformのbackendとしてS3を使う"
date: 2024-08-31T11:55:26+09:00
draft: false
categories: ["terraform"]
---

tfstateをローカルで作成してから、S3に移すというのを試してみたいと思います。

## tfstateをローカルで作成する

まずは`test`というディレクトリの中に`local.tf`というファイルを作成します。

```
% tree .
.
└── test
    └── local.tf
```

`local.tf`には以下のように`local_file`を定義します。

```
resource "local_file" "foo" {
  content  = "Hello, World!"
  filename = "/tmp/foo"
}
```

参考: https://registry.terraform.io/providers/hashicorp/local/latest/docs/resources/file


そして`terraform init`, `terraform plan`, `terraform apply`を順次実行していきます。

```
% terraform init
% terraform plan
% terraform apply
```

そうすると、ローカルには`terraform.tfstate`というファイルが作成されます。
```
% tree .
.
└── test
    ├── local.tf
    └── terraform.tfstate
```


## tfstateをs3に移す

### IAMユーザーの追加

まず初めにIAMユーザーを追加します。

今回はAWSコンソールから`terraformer`というユーザーを作成し、`AmazonS3FllAccess`の権限を付与します。

![](/images/terraform-backend/s3-full-access.png)

その後、「アクセスキーを作成」からCLI用のアクセスキーを作成します。

作成されたアクセスキー、シークレットキーをローカルの`~/.aws/config`に以下のように記述します。

```
[profile terraformer]
region = ap-northeast-1
aws_access_key_id = XXXXX
aws_secret_access_key = XXXXX
```

### S3バケットの作成

AWSコンソールから`kkato-terraform-state`というS3バケットを作成します。

### Terraform側の設定

`main.tf`というファイルを作成し、backendとproviderの設定をします。

```
terraform {
  backend "s3" {
    bucket = "kkato-terraform-state"
    key = "test/terraform.tfstate"
    region = "ap-northeast-1"
    profile = "terraformer"
  }
}

provider "aws" {
  region = "ap-northeast-1"
}
```

その後`terraform init`を実行すると、`tfstate`をS3に移すことができます。

```
% terraform init

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly
  configured "s3" backend. Do you want to copy this state to the new "s3"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes


Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Reusing previous version of hashicorp/local from the dependency lock file
- Finding latest version of hashicorp/aws...
- Using previously-installed hashicorp/local v2.5.1
- Installing hashicorp/aws v5.65.0...
- Installed hashicorp/aws v5.65.0 (signed by HashiCorp)

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

不要になったローカルの`tfstate`ファイルは削除します。

```
% rm terraform.tfstate
% rm terraform.tfstate.backup
```
