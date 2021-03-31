---
title: "Amazon Linux2にJenkinsをインストールする"
date: "2021-03-17"
tags: ["AWS", "Jenkins"]
author: "bokyung"
---

ローカル環境でJenkinsを試しましたが、エラーになるケールがあったため、検証のために、Amazon Linux2にJenkinsをインストールしてみました。

## 構築するJenkinsの環境
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI名 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## EC2インスタンス作成及びJenkinsインストール
### EC2インスタンス作成ととセキュリティグループ設定
* 一番上に表示されているAMIを利用して作成しました。
{{< img src="/images/2021/0317/ec2_ja.png" alt="ec2_ja" link="/images/2021/0317/ec2_ja.png">}}

* Jenkinsアクセス用の8080ポートと、SSH接続用の22ポートを設定します。
{{< img src="/images/2021/0317/ec2_sg_ja.png" alt="ec2_sg_ja" link="/images/2021/0317/ec2_sg_ja.png">}}


### Jenkinsインストール
* [Jenkins公式サイトのインストール方法](https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos)の順番通りに実行。

```
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade
sudo yum install jenkins java-1.8.0-openjdk-devel.x86_64
sudo systemctl daemon-reload
```

### Jenkins 実行
```
sudo systemctl start jenkins
```

サービス実行状態を確認し、active状態になっていることを確認します。
{{< img src="/images/2021/0317/jenkins_status.png" alt="jenkins_status" link="/images/2021/0317/jenkins_status.png">}}

## Jenkins 初期設定
* Jenkinsへアクセスします。http://xxxxxxxxxx:8080/
* 初期パスワードを入力します。
{{< img src="/images/2021/0317/jenkins_init.png" alt="jenkins_init" link="/images/2021/0317/jenkins_init.png">}}

* プラグインをインストールします。
{{< img src="/images/2021/0317/jenkins_custom.png" alt="jenkins_custom" link="/images/2021/0317/jenkins_custom.png">}}

* 初期ユーザーを登録します。
{{< img src="/images/2021/0317/jenkins_start_user_ja.png" alt="jenkins_start_user_ja" link="/images/2021/0317/jenkins_start_user_ja.png">}}


* インスタンスURLを設定します。
{{< img src="/images/2021/0317/jenkins_start_instance.png" alt="jenkins_start_instance" link="/images/2021/0317/jenkins_start_instance.png">}}

* 設定が完了しました。
{{< img src="/images/2021/0317/jenkins_start.png" alt="jenkins_start" link="/images/2021/0317/jenkins_start.png">}}

* メイン画面が表示されました。
{{< img src="/images/2021/0317/jenkins_top_ja.png" alt="jenkins_top_ja" link="/images/2021/0317/jenkins_top_ja.png">}}



