# Terraform / IaC

**インフラをコードで管理する**（Infrastructure as Code）ツールです。「AWS にサーバーを立てる」「GCP にデータベースを作る」という操作をクリックではなくコードで書き、Git で管理します。同じ構成を何度でも再現でき、変更を差分として確認できます。

---

## はじめて読む人へ

「インフラを手動で作る → 誰が何を作ったかわからなくなる → 本番と開発環境がズレる」という問題を IaC が解決します。Terraform は AWS・GCP・Azure すべてに対応した最も広く使われる IaC ツールです。

### 読む前に押さえること

- [クラウド・インフラ](クラウド-インフラ.md) の VM・コンテナ・クラウドサービスの概念
- [Git](Git.md) のバージョン管理の基本
- [CI/CD](CI-CD.md) を読んでいると GitHub Actions との組み合わせが理解しやすい

### 読み終えたら説明できること

- IaC が「手動インフラ管理」より優れている理由を説明できる
- Provider・Resource・State の関係を説明できる
- `terraform plan` と `terraform apply` の違いを説明できる

---

## なぜ IaC が必要か

!!! info ""
    **手動管理の問題**

    開発者A: コンソールでサーバーを作成
    開発者B: 「このサーバーどうやって作ったの？」→ 不明
    本番環境: 誰かが手動で設定変更 → ドキュメントと乖離
    障害発生: 同じ構成を再現しようとしても手順が不明

    **IaC（Terraform）**

    git log で「誰が・いつ・何を」変更したか一目瞭然
    terraform plan でデプロイ前に変更内容を確認
    同じコードから開発・ステージング・本番を再現
    PR レビューでインフラ変更をコードレビューできる
---

## インストールと基本構成

```bash
# Mac
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# バージョン確認
terraform version
```

Terraform のファイル構成：

!!! info ""
    ```text
    infra/
    ├── main.tf          # メインのリソース定義
    ├── variables.tf     # 変数定義
    ├── outputs.tf       # 出力値
    ├── providers.tf     # クラウドプロバイダー設定
    └── terraform.tfvars # 変数の実際の値（.gitignore に追加）
    ```
---

## 基本概念

### Provider

どのクラウドに対してリソースを作るかを指定します。

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"  # 東京リージョン
}
```

### Resource

作成するインフラのリソースです。

```hcl
# main.tf
# S3 バケットを作成
resource "aws_s3_bucket" "website" {
  bucket = "my-website-bucket-2026"
}

# バケットのパブリックアクセス設定
resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id  # 上のリソースを参照

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Variables（変数）

環境ごとに異なる値を変数で管理します。

```hcl
# variables.tf
variable "environment" {
  description = "デプロイ環境"
  type        = string
  default     = "development"
}

variable "instance_type" {
  description = "EC2 インスタンスタイプ"
  type        = string
}

variable "db_password" {
  description = "DB パスワード"
  type        = string
  sensitive   = true  # ログに表示しない
}
```

```hcl
# terraform.tfvars（.gitignore に追加すること）
environment   = "production"
instance_type = "t3.micro"
db_password   = "your-secret-password"
```

```hcl
# 変数を参照する
resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags = {
    Environment = var.environment
  }
}
```

### Outputs（出力）

作成されたリソースの情報（IP アドレスなど）を出力します。

```hcl
# outputs.tf
output "s3_bucket_name" {
  value       = aws_s3_bucket.website.bucket
  description = "作成された S3 バケット名"
}

output "website_url" {
  value = "https://${aws_s3_bucket.website.bucket}.s3.amazonaws.com"
}
```

---

## 基本コマンド

```bash
# 初期化（プロバイダーをダウンロード）
terraform init

# 実行計画の確認（実際には何も変更しない）
terraform plan

# 実際に適用（+ → 作成、~ → 更新、- → 削除）
terraform apply

# 特定のリソースだけ適用
terraform apply -target=aws_s3_bucket.website

# リソースを削除
terraform destroy

# 現在の状態を確認
terraform show

# 書式を自動修正
terraform fmt
```

`terraform plan` の出力例：

```
Plan: 2 to add, 0 to change, 0 to destroy.

+ resource "aws_s3_bucket" "website" {
    bucket = "my-website-bucket-2026"
  }

+ resource "aws_s3_bucket_public_access_block" "website" {
    block_public_acls = true
    ...
  }
```

`+` が追加、`~` が変更、`-` が削除です。`apply` 前に必ず `plan` で確認します。

---

## State（状態ファイル）

Terraform は作成したリソースの状態を `terraform.tfstate` に記録します。

!!! info ""
    ```text
    terraform.tfstate: どのリソースが作成されているかの「現実の状態」
    main.tf:           「あるべき状態」の定義

    terraform apply 時: あるべき状態 ↔ 現実の状態 の差分を計算して適用
    ```
**チームで使うときはリモートバックエンドが必須**（ローカルの .tfstate をコミットしてはいけない）。

```hcl
# S3 をバックエンドとして使う例
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "ap-northeast-1"
  }
}
```

---

## 実践例：Web サーバーの構築

```hcl
# main.tf
# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "${var.environment}-vpc" }
}

# サブネット
resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-1a"
}

# セキュリティグループ
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
  name   = "${var.environment}-web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 インスタンス
resource "aws_instance" "web" {
  ami           = "ami-0d52744d6551d851e"  # Amazon Linux 2023
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y nginx
    systemctl start nginx
  EOF

  tags = { Name = "${var.environment}-web" }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

## GitHub Actions との組み合わせ

PR 時に `terraform plan` を自動実行し、結果をコメントとして表示します。

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - "infra/**"

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1

      - run: terraform init
      - run: terraform fmt -check        # 書式チェック
      - run: terraform validate          # 構文チェック
      - name: Terraform Plan
        run: terraform plan -no-color 2>&1 | tee plan.txt

      - name: PR にプランをコメント
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infra/plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `\`\`\`\n${plan}\n\`\`\``
            });
```

---

## 確認問題

1. `terraform plan` と `terraform apply` の違いを説明してください。
2. `terraform.tfstate` をチームで共有するためにリモートバックエンドが必要な理由を説明してください。
3. `terraform.tfvars` を `.gitignore` に追加すべき理由を説明してください。

---

## 関連ページ

- [クラウド・インフラ](クラウド-インフラ.md) — AWS/GCP のサービス概要
- [Docker](Docker.md) — コンテナとの組み合わせ
- [Kubernetes](Kubernetes.md) — k8s クラスターを Terraform で管理
- [CI/CD](CI-CD.md) — GitHub Actions との統合
- [セキュリティ基礎](セキュリティ.md) — tfvars の秘密情報管理
