# AWS上でのLAMP環境構築手順書

## 概要

本手順書は、AWS(Amazon Web Services)上にUbuntu Serverベースの仮想サーバーを構築し、Apache・PHP・MySQLからなるLAMP環境を構築するための手順をまとめたものである。手動構築の手順(第1部)と、Terraformによるコード化・自動化の手順(第2部)の両方を記載する。

対象読者は、AWSアカウントを保有し、基本的なLinuxコマンド操作の知識がある者を想定する。

---

## 第1部:手動構築

### 1. 事前準備

#### 1.1 AWSアカウントの確認

- AWSアカウントが作成済みであること(Paidプラン推奨。Freeプランは6ヶ月または200USDのクレジット消化で自動的にアカウントが閉じるため、継続利用の予定がある場合はPaidプランを選択する)
- コスト予算アラート(AWS Budgets)が設定済みであること。推奨設定は月次コスト予算1USD、しきい値85%/100%/予測100%の3段階通知

**チェックポイント(1)**: AWS Budgetsのダッシュボードで、予算のヘルスステータスが「健康」であることを確認する。

#### 1.2 リージョンの選択

画面右上のリージョン表示を「アジアパシフィック(東京)ap-northeast-1」に切り替える。日本国内からのアクセスの場合、レイテンシとコストの両面で東京リージョンが適切である。

---

### 2. EC2インスタンスの起動

#### 2.1 AMI(Amazon Machine Image)の選択

EC2の「インスタンスを起動」画面にて、AMI検索欄に「Ubuntu」と入力し、候補から「Ubuntu Server 26.04 LTS」を明示的に選択する。

> **注意**: AWSのデフォルト選択はAmazon Linuxになっている場合がある。既存のUbuntu運用ノウハウ(apt系コマンド)を活かすためには、AMIの選択を必ず目視で確認すること。誤ってAmazon Linuxのまま起動すると、ユーザー名が`ubuntu`ではなく`ec2-user`になり、パッケージ管理コマンドも`apt`ではなく`dnf`/`yum`に変わる。

#### 2.2 インスタンスタイプの選択

`t3.micro`(2 vCPU、1GiBメモリ)を選択する。学習・検証用途としては十分なスペックであり、無料利用枠の対象にもなりやすい。

#### 2.3 キーペアの作成

「新しいキーペアの作成」を選択し、キーペアタイプは「RSA」、プライベートキー形式は「.pem」を選択して作成する。ダウンロードされる`.pem`ファイルは再ダウンロード不可のため、確実な場所に保管する。

#### 2.4 起動

「インスタンスを起動」をクリックし、起動完了(ステータスが「実行中」)を待つ。

**チェックポイント(2)**: インスタンス詳細画面の「AMIの場所」欄が`ubuntu/images/...`のような表記になっていることを確認する(`amazon/al2023-...`となっていた場合はAMI選択ミス)。

---

### 3. セキュリティグループの設定

#### 3.1 インバウンドルールの編集

対象インスタンスのセキュリティグループを開き、インバウンドルールを以下の通り設定する。

| タイプ | ポート範囲 | ソース | 用途 |
|---|---|---|---|
| SSH | 22 | マイIP(自分のIPアドレス/32) | 管理者のみがSSH接続可能にする |
| HTTP | 80 | Anywhere-IPv4(0.0.0.0/0) | 一般公開するWebサービス用 |

> **注意**: ソース欄で「マイIP」や「Anywhere-IPv4」を選択する際は、必ず検索欄に表示される候補をクリックして選択すること。文字列として手入力すると、実際のCIDR表記に変換されずエラーになる。

**チェックポイント(3)**: インバウンドルールが2件登録されており、SSHのソースが世界中(0.0.0.0/0)に開放されていないことを確認する。

---

### 4. SSH接続

#### 4.1 接続コマンド

Windows環境の場合、PowerShellから以下のコマンドで接続する。

```powershell
ssh -i "キーペアファイル名.pem" ubuntu@<パブリックIPアドレス>
```

初回接続時は、ホストの信頼性確認(fingerprint確認)が表示されるため、`yes`と入力する。

> **注意**: `.pem`ファイルの保存場所が、OneDrive等でリダイレクトされたデスクトップフォルダである場合、`cd ~\Desktop`のような単純な移動コマンドでは見つからないことがある。`Get-ChildItem -Recurse -Filter "*.pem"`で実ファイルの場所を検索するとよい。

**チェックポイント(4)**: `ubuntu@ip-xxx-xxx-xxx-xxx:~$`のプロンプトが表示され、正常にログインできることを確認する。

---

### 5. Apache・PHP・MySQLの構築

#### 5.1 パッケージのインストール

```bash
sudo apt update
sudo apt install apache2 -y
sudo apt install php libapache2-mod-php php-mysql -y
sudo systemctl restart apache2
sudo apt install mysql-server -y
```

#### 5.2 動作確認用ファイルの作成

このステップは、5.1でPHPのインストールとApacheの再起動が完了した直後、SSH接続を維持したまま実行する。MySQLのインストール(5.3)より前に行っても後に行っても動作に支障はないが、本手順書では「PHPが正しく動くこと」を先に確認してから次に進む順序とする。

```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/sample.php
```

作成後、いったんブラウザで `http://<パブリックIPアドレス>/sample.php` にアクセスし、PHPバージョン情報が表示されることを確認してから、5.3に進む。

**チェックポイント(5)**: `sample.php`にアクセスし、PHPのバージョン情報ページが表示されることを確認する。この時点では`mysqli`・`pdo_mysql`拡張の項目も表示されるが、MySQL自体はまだ未インストールのため、実際のデータベース接続はまだ確認できない(5.3完了後にチェックポイント6で改めて確認する)。

#### 5.3 MySQLユーザーの作成

MySQLインストール後、まず管理者としてログインする。

```bash
sudo mysql -u root
```

`mysql>` プロンプトに変わったら、以下を実行してユーザーとデータベースを作成する。

```sql
CREATE USER 'ユーザー名'@'localhost' IDENTIFIED BY 'パスワード';
GRANT ALL ON testdb.* TO 'ユーザー名'@'localhost';
CREATE DATABASE testdb;
SHOW DATABASES;
```

`exit`でrootユーザーから抜けた後、作成したユーザーで再ログインできるか確認する。

```bash
mysql -u ユーザー名 -p
```

パスワードを求められたら、作成時に指定したパスワードを入力する。

**チェックポイント(6)**: 作成したユーザーで`mysql -u ユーザー名 -p`によるログインができ、`SHOW DATABASES;`で`testdb`が表示されることを確認する。

---

### 6. 作業終了時の後片付け

継続利用の予定がある場合は「インスタンスの停止」を選択する(ディスク保存料金のみ発生)。今後利用しない場合は「インスタンスの終了」を選択し、リソースを完全に削除する。

> **注意**: 停止と終了(Terminate)は別物である。停止中もEBSボリュームの保存料金はわずかに発生し続けるため、完全に不要な場合は終了させること。

---

## 第2部:Terraformによるコード化

### 7. Terraformのセットアップ

#### 7.1 インストール

```powershell
winget install HashiCorp.Terraform
```

インストール後、PATH環境変数の反映のためシェルを再起動する。

#### 7.2 IAMユーザーとアクセスキーの準備

Terraform専用のIAMユーザー(例: `terraform-user`)を作成し、`AmazonEC2FullAccess`ポリシーをアタッチする。アクセスキーを発行し、以下のように環境変数として設定する(コード上には記述しない)。

```powershell
$env:AWS_ACCESS_KEY_ID="アクセスキーID"
$env:AWS_SECRET_ACCESS_KEY="シークレットアクセスキー"
$env:AWS_DEFAULT_REGION="ap-northeast-1"
```

> **重要**: アクセスキー・シークレットアクセスキーは、チャットやコード、Gitリポジトリに一切残さないこと。誤って共有した場合は、直ちに該当キーを無効化・削除し、新しいキーを再発行する。

**チェックポイント(7)**: `$env:AWS_ACCESS_KEY_ID.Length`が20、`$env:AWS_SECRET_ACCESS_KEY.Length`が40であることを確認する(コピー時の文字化け・余分な文字混入の検知に有効)。

---

### 8. Terraformコードの作成と適用

#### 8.1 コード例(`main.tf`)

第1部で手動構築した内容を、以下のようにコード化する。コード中の`自分のIPアドレス/32`は手入力が前提となる箇所であり、環境によって値が変わる。以下のいずれかの方法で、現在使用しているグローバルIPアドレスを事前に確認しておくこと。

- ブラウザで `https://checkip.amazonaws.com` にアクセスし、表示されたIPアドレスを使う
- AWSコンソールのセキュリティグループ編集画面で、ソース欄に「マイIP」を選択した際に自動入力される値を確認し、それを控えておく(第1部 3.1参照)

このIPアドレスは、Wi-Fi再接続や別のネットワークへの切り替えによって変わることがある。`terraform apply`前に必ず現在の値と一致しているか再確認すること。

```hcl
provider "aws" {
  region = "ap-northeast-1"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"]
  }
}

variable "db_password" {
  description = "MySQL admin password"
  type        = string
  sensitive   = true
}

resource "aws_security_group" "web_sg" {
  name        = "terraform-web-sg"
  description = "Allow SSH from my IP and HTTP from anywhere"

  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["自分のIPアドレス/32"]
  }

  ingress {
    description = "HTTP from anywhere"
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

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t3.micro"
  key_name               = "キーペア名"
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  user_data = <<-EOF
              #!/bin/bash
              apt update -y
              apt install -y apache2 php libapache2-mod-php php-mysql mysql-server
              mysql -e "CREATE USER 'admin'@'localhost' IDENTIFIED BY '${var.db_password}';"
              mysql -e "GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost';"
              systemctl start apache2
              EOF

  tags = {
    Name = "terraform-web-server"
  }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

#### 8.2 実行コマンド

```powershell
terraform init    # プロバイダプラグインの初期化(初回のみ)
terraform plan    # 適用内容の事前確認(何が作成/変更/削除されるか)
terraform apply -var="db_password=安全なパスワード"   # 実際に適用
```

`apply`実行後、コンソールに`yes`と入力して確定する。

**チェックポイント(8)**: `Apply complete! Resources: 2 added, 0 changed, 0 destroyed.`のようなメッセージと、`public_ip`の出力値が表示されることを確認する。

#### 8.3 動作確認

出力された`public_ip`に対して、ブラウザで以下にアクセスし、Apache・PHPが自動構築されていることを確認する。

- `http://<public_ip>/` → Apacheデフォルトページ
- `http://<public_ip>/sample.php` → PHP情報ページ

> **補足**: `user_data`はインスタンスの初回起動時にのみ実行される。既存の起動中インスタンスに対して`user_data`だけを変更しても、内部のスクリプトは再実行されない。変更を反映するには `terraform apply -replace="aws_instance.web"` でインスタンスを再作成する必要がある。

#### 8.4 環境の削除

`terraform destroy`は、`main.tf`に定義されている**すべてのリソースを問答無用で削除するコマンド**である。一部だけを選んで消す、といった操作はできないため、実行前には以下の点に注意すること。

- **データは復元できない**: インスタンス内に保存したファイルやMySQLのデータベースは、インスタンスの削除と同時に完全に失われる。停止(Stop)とは異なり、後から再開して中身を見ることはできない。重要なデータがある場合は、事前にバックアップ(スナップショット取得やダンプの取得)を行うこと。
- **実行前に影響範囲を確認する**: 何が削除されるか不安な場合は、`terraform destroy`の前に以下のコマンドで削除対象を事前確認できる。

  ```powershell
  terraform plan -destroy
  ```

- **複数人で同じ状態を触っている場合は要注意**: チームで同じ`terraform.tfstate`を共有している環境では、自分の意図しないタイミングで他の担当者の作業対象まで削除してしまう恐れがある。実務では、destroyの実行権限やタイミングをチームで取り決めておく。

削除の実行は以下の通り。

```powershell
terraform destroy
```

確認プロンプトに`yes`と入力すると、`main.tf`に定義された全リソースが削除される。コード自体(`main.tf`)は手元に残るため、必要になれば`terraform apply`一発で同一構成(新しいリソースIDで)を再現できる。

---

## 9. セキュリティ上の留意事項(生成AIによるコード生成時のレビュー観点)

生成AIにIaCコードを生成させる場合、以下の観点で人間によるレビューを必ず行うこと。

1. **認証情報の直書きがないか**: パスワードやAPIキーがコード内にハードコードされていないか。`variable`ブロックと`sensitive = true`を用い、値は環境変数やコマンドライン引数で渡す。
2. **ネットワーク開放範囲が適切か**: SSH等の管理用ポートが`0.0.0.0/0`(全世界)に開放されていないか。管理者のIPアドレスに限定する。
3. **リソースへのアクセス手段が揃っているか**: キーペアの指定漏れなど、後から接続できなくなる設定ミスがないか。
4. **ハードコードされた値の陳腐化リスク**: AMI IDなど時間経過で無効になりうる値は、`data`ブロックによる動的取得に置き換える。
5. **状態ファイル(`terraform.tfstate`)の扱い**: 状態ファイルには機密情報が平文で記録されることがあるため、Gitには絶対にコミットせず、暗号化されたリモートバックエンドで管理する。

---

## 改訂履歴

| 日付 | 内容 |
|---|---|
| 2026-07-18 | 初版作成(AWS手動構築・Terraform化・生成AIコードレビューの一連の作業をもとに作成) |
