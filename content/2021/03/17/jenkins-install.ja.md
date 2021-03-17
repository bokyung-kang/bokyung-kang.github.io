---
title: "Amazon Linux2にJenkinsをインストールする"
date: "2021-03-17"
tags: ["AWS", "Jenkins"]
author: "bokyung"
---

ローカル環境でJenkinsを試しましたが、エラーになるケールがあったため、検証のために、Amazon Linux2にJenkinsをインストールしてみました。

## EC2インスタンス作成及びJenkinsインストール
### EC2インスタンス作成ととセキュリティグループ設定
* 一番上に表示されているAMIを利用して作成しました。
![ec2](/images/2021/0317/ec2_ja.png)

* Jenkinsアクセス用の8080ポートと、SSH接続用の22ポートを設定します。
![ec2_sg](/images/2021/0317/ec2_sg_ja.png)

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
![jenkins_status](/images/2021/0317/jenkins_status.png)

## Jenkins 初期設定
* Jenkinsへアクセスします。http://xxxxxxxxxx:8080/
* 初期パスワードを入力します。
![jenkins_init](/images/2021/0317/jenkins_init.png)

* プラグインをインストールします。
![jenkins_custom](/images/2021/0317/jenkins_custom.png)

* 初期ユーザーを登録します。
![jenkins_start_user](/images/2021/0317/jenkins_start_user_ja.png)

* インスタンスURLを設定します。
![contexts](/images/2021/0317/jenkins_start_instance.png)

* 設定が完了しました。
![contexts](/images/2021/0317/jenkins_start.png)

* メイン画面が表示されました。
![contexts](/images/2021/0317/jenkins_top_ja.png)


