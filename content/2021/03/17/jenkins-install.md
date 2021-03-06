---
title: "Amazon Linux2에 Jenkins 설치하기"
date: "2021-03-17"
tags: ["AWS", "Jenkins"]
author: "bokyung"
---

로컬 도커환경에서 Jenkins를 테스트 하던 중에 에러가 나는 경우가 있어서 검증을 위해서 Amazon Linux2에 Jenkins를 설치 해 보았습니다.

## 설치한 Jenkins 환경
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI 이름 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## EC2 인스턴스 생성 및 Jenkins설치
### EC2 인스턴스 생성 및 보안그룹 설정
* 맨위에 표시되는 AMI로 생성했습니다.
{{< img src="/images/2021/0317/ec2.png" alt="ec2" link="/images/2021/0317/ec2.png">}}


* 젠킨스 접속용 8080포트와 SSH접속용 22번 포트를 설정합니다.
{{< img src="/images/2021/0317/ec2_sg.png" alt="ec2_sg" link="/images/2021/0317/ec2_sg.png">}}


### Jenkins 설치
* [Jenkins 공홈 설치 방법](https://www.jenkins.io/doc/book/installing/linux/#red-hat-centos)에 나와있는 순서대로 설치하였습니다.
```
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade
sudo yum install jenkins java-1.8.0-openjdk-devel.x86_64
sudo systemctl daemon-reload
```
### Jenkins 실행
```
sudo systemctl start jenkins
```

서비스 실행 상태를 확인하여, active상태인 것을 확인합니다.
{{< img src="/images/2021/0317/jenkins_status.png" alt="jenkins_status" link="/images/2021/0317/jenkins_status.png">}}

## Jenkins 초기 설정
* Jenkins에 접속합니다. http://xxxxxxxxxx:8080/
* 초기패스워드를 화면에 입력합니다.
{{< img src="/images/2021/0317/jenkins_init.png" alt="jenkins_init" link="/images/2021/0317/jenkins_init.png">}}

* 추천플러그인을 설치합니다.
{{< img src="/images/2021/0317/jenkins_custom.png" alt="jenkins_custom" link="/images/2021/0317/jenkins_custom.png">}}

* 초기유저를 등록합니다.
{{< img src="/images/2021/0317/jenkins_start_user.png" alt="jenkins_start_user" link="/images/2021/0317/jenkins_start_user.png">}}

* 인스턴스 접속주소를 등록합니다.
{{< img src="/images/2021/0317/jenkins_start_instance.png" alt="jenkins_start_instance" link="/images/2021/0317/jenkins_start_instance.png">}}

* 설정이 완료되었습니다.
{{< img src="/images/2021/0317/jenkins_start.png" alt="jenkins_start" link="/images/2021/0317/jenkins_start.png">}}

* 메인화면이 표시되었습니다.
{{< img src="/images/2021/0317/jenkins_top.png" alt="jenkins_top" link="/images/2021/0317/jenkins_top.png">}}



