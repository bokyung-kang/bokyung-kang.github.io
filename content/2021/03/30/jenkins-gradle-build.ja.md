---
title: "Jenkins環境でSpring BatchプロジェクトをGradleビルドする"
date: "2021-03-30"
tags: ["AWS", "Jenkins", "Spring", "gradle"]
author: "bokyung"
---

Spring BatchのJob Runnerを調査していましたが、その中でJenkinsでbatchプロジェクトをgradleビルドをしてみました。

## Jenkinsの環境
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI名 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## 認証情報登録
### Github ID/PASSWORD認証方法
* JenkinsでRepository関連作業ができるようにGithub上で`Personal access tokens`を発行します。 ( Settings > Developer settings > Personal access tokens )
{{< img src="/images/2021/0330/gihhub-personal-access-tokens.png" alt="gihhub-personal-access-tokens" link="/images/2021/0330/gihhub-personal-access-tokens.png">}}

* Manage Jenkins > Manage Credentials > domainにあるglobalをクリックします。
{{< img src="/images/2021/0330/global-credential.png" alt="global-credential" link="/images/2021/0330/global-credential.png">}}
* Kindは`Username with password`を選び、パスワード欄には`Personal access tokens`値を入力し認証情報を追加します。
{{< img src="/images/2021/0330/ssh-credential.png" alt="ssh-credential" link="/images/2021/0330/ssh-credential.png">}}

### Github SSH 認証方法
* SSH鍵を作成します。
{{< img src="/images/2021/0330/generate-ssh-key.png" alt="generate-ssh-key" link="/images/2021/0330/generate-ssh-key.png">}}
{{< img src="/images/2021/0330/jenkins-ssh-key.png" alt="jenkins-ssh-key" link="/images/2021/0330/jenkins-ssh-key.png">}}

* Githubプロジェクト設定の`Deploy keys`に公開鍵を追加します。
{{< img src="/images/2021/0330/deploy-keys.png" alt="deploy-keys" link="/images/2021/0330/deploy-keys.png">}}

* Kindは`SSH Username with private key`を選び、`Private Key > Enter directly`に秘密鍵を追加します。
{{< img src="/images/2021/0330/ssh-credential.png" alt="ssh-credential" link="/images/2021/0330/ssh-credential.png">}}


### Gradle, JDK設定
* プロジェクトで利用するGradleとJDKを `Manage Jenkins > System Configuration > Global Tool Configuration`にて設定します。一つのバージョンのみ設定した場合は、各JOBでデフォルトとして設定されます。複数のバージョンが登録されている場合は、JOB設定時バージョンを選べられます。
{{< img src="/images/2021/0330/jenkins-global-tool-configuration.png" alt="jenkins-global-tool-configuration" link="/images/2021/0330/jenkins-global-tool-configuration.png">}}

### Job登録
* `New Item > Freestyle project`を選び、JOBを登録します。github access tokens認証の場合は、Repository URLに `git@github.com:bokyung-kang/batch-test`を入力しCredentialsはgithub認証を登録したUsernameを選択します。
{{< img src="/images/2021/0330/build-job-github.png" alt="build-job-github" link="/images/2021/0330/build-job-github.png">}}

* ssh認証の場合はRepository URLに `git@github.com:bokyung-kang/batch-test`を入力しCredentialsはssh認証を登録したUsernameを選択します。
{{< img src="/images/2021/0330/build-job-ssh.png" alt="build-job-ssh" link="/images/2021/0330/build-job-ssh.png">}}

* build script
  ```
  chmod +x gradlew

  # testを実行しないようにassembleを実行しました。
  ./gradlew assemble

  # deploy-testへjarをコピー。
  mkdir -p $JENKINS_HOME/workspace/deploy-test
  cp $WORKSPACE/build/libs/*.jar $JENKINS_HOME/workspace/deploy-test/.
  ```

### Gradle Build
* ビルドを実行し、Console Outputを見てみます。
```
[batch-test] $ /bin/sh -xe /tmp/jenkins568576160326820555.sh
+ chmod +x gradlew
+ ./gradlew assemble
Starting a Gradle Daemon (subsequent builds will be faster)
> Task :compileKotlin
> Task :compileJava NO-SOURCE
> Task :processResources
> Task :classes
> Task :bootJarMainClassName
> Task :bootJar
> Task :inspectClassesForKotlinIC
> Task :jar SKIPPED
> Task :assemble

BUILD SUCCESSFUL in 33s
5 actionable tasks: 5 executed
+ mkdir -p /var/lib/jenkins/workspace/deploy-test
+ cp /var/lib/jenkins/workspace/batch-test/build/libs/batch-0.0.1-SNAPSHOT.jar /var/lib/jenkins/workspace/deploy-test/.
Finished: SUCCESS
```

* `$JENKINS_HOME/workspace/deploy-test/`にjarファイルがコピーされていることが確認できます。
{{< img src="/images/2021/0330/build-jar.png" alt="build-jar" link="/images/2021/0330/build-jar.png">}}

