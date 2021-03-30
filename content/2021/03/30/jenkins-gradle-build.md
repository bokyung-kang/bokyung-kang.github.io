---
title: "Jenkins에서 Spring Batch 프로젝트를 Gradle 빌드하기"
date: "2021-03-30"
tags: ["AWS", "Jenkins", "Spring", "gradle"]
author: "bokyung"
---

스프링 배치 Job Runner를 조사하던 중에 Jenkins에서 gradle빌드해 보았습니다.

## Jenkins 환경
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI 이름 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## 인증정보 등록
### Github 아이디/패스워드 인증 방법
* Jenkins에서 Repository관련 작업이 가능하도록 Github에서 `Personal access tokens`를 발행합니다. ( Settings > Developer settings > Personal access tokens )
![global-credential](/images/2021/0330/gihhub-personal-access-tokens.png)

* Manage Jenkins > Manage Credentials > domain에 있는 global을 클릭합니다.
![global-credential](/images/2021/0330/global-credential.png)

* Kind는 `Username with password`를 선택하고 패스워드란에 `Personal access tokens`값을 입력하고 인증정보를 추가합니다.
![global-credential](/images/2021/0330/github-credential.png)

### Github SSH 인증 방법
* SSH키를 생성합니다.
![generate-ssh-key](/images/2021/0330/generate-ssh-key.png)
![jenkins-ssh-key](/images/2021/0330/jenkins-ssh-key.png)

* Github 프로젝트 설정에서 `Deploy keys`에 공개키를 추가합니다.
![deploy-keys](/images/2021/0330/deploy-keys.png)

* Kind는 `SSH Username with private key`를 선택하고 `Private Key > Enter directly` 에 비밀키를 추가합니다.
![deploy-keys](/images/2021/0330/ssh-credential.png)

### Gradle, JDK설정
* 프로젝트에서 사용할 Gradle과 JDK를 `Manage Jenkins > System Configuration > Global Tool Configuration`에서 설정합니다. 한가지 버전만 설정한 경우 각 JOB에서 디폴트로 설정됩니다. 복수개 버전이 등록된 경우에는 JOB설정시 버전선택이 가능합니다.
![deploy-keys](/images/2021/0330/jenkins-global-tool-configuration.png)

### Job등록
* `New Item > Freestyle project`을 선택해서 Job을 등록합니다. github access tokens으로 인증할 경우에는 Repository URL에 `git@github.com:bokyung-kang/batch-test`을 입력하고 Credentials에서 github인증을 등록한 Username을 선택합니다.
![global-credential](/images/2021/0330/build-job-github.png)

* ssh인증할 경우에는 Repository URL에 `git@github.com:bokyung-kang/batch-test`을 입력하고 Credentials에서 ssh을 등록한 Username을 선택합니다.
![global-credential](/images/2021/0330/build-job-ssh.png)

* build script
  ```
  chmod +x gradlew

  # test를 실행안하려고 저는 assemble을 실행했습니다
  ./gradlew assemble

  # deploy-test에  jar복사
  mkdir -p $JENKINS_HOME/workspace/deploy-test
  cp $WORKSPACE/build/libs/*.jar $JENKINS_HOME/workspace/deploy-test/.
  ```

### Gradle Build
* 빌드를 실행합니다.
![global-credential](/images/2021/0330/gradle-build.png)

* `$JENKINS_HOME/workspace/deploy-test/`에 jar파일이 복사되었네요.
![global-credential](/images/2021/0330/build-jar.png)

