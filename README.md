# aws-lamp-setup-guide

AWS上にUbuntu Server + Apache + PHP + MySQLのLAMP環境を構築するための手順書です。手動構築(第1部)と、Terraformによるコード化(第2部)の両方の手順を、Document as Code(DaC)の考え方に基づいてMarkdownでまとめています。

## 内容

- 事前準備(AWSアカウント・予算アラート・リージョン選択)
- EC2インスタンスの起動、セキュリティグループの設定、SSH接続
- Apache・PHP・MySQLの構築と動作確認
- Terraformによるインフラのコード化と自動プロビジョニング(user_data)
- 生成AIが生成したIaCコードをレビューする際の観点

各手順には、実際に読んで再現できるよう「チェックポイント」形式で確認項目を明記しています。

## 関連リポジトリ

- [aws-lamp-terraform](https://github.com/saki-nya1539/aws-lamp-terraform) : 本手順書のTerraformコード実装
- [iac-ai-code-review](https://github.com/saki-nya1539/iac-ai-code-review) : 生成AIが書いたIaCコードのレビュー・改善記録
