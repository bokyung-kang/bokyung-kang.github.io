---
title: "AWS SAM을 이용해서 Lambda함수 배포하기"
date: "2021-05-17"
tags: ["AWS", "AWS SAM", "AWS Lambda", "AWS S3"]
author: "bokyung"
---

이전 글에서 [CloudFront와 Lambda@Edge를 이용해서 이미지 리사이징하기]({{< ref "/2021/05/14/lambda-edge-resize" >}})를 정리해 보았는데요.
AWS SAM을 이용해서 CloudFormation으로 Lambda함수를  배포하는 방법을 정리해 보았습니다.

## 배포순서
### SAM CLI설치
[Install SAM CLI](https://aws.amazon.com/ko/serverless/SAM/)

저는 `SAM CLI, version 1.23.0`을 이용했습니다
### template.yaml작성
CloudFormation기반의 템플릿(template.yaml)을 작성합니다.
```shell
.
├── index.js
├── node_modules/
├── template.yaml
├── package-lock.json
└── package.json
```
{{< gist bokyung-kang aa6452d3d4f559c4d77ac11fa5bdc876 >}}

### SAM deploy을 이용하여 코드 패키징과 배포
```shell
SAM deploy \
    --template-file template.yaml \
    # Lambda@edge인 경우는 us-east-1리전에 만든 Bucket이 필요합니다.
    --s3-bucket <zip파일을 업로드 할  Bucket name> \
    --s3-prefix SAM \
    --stack-name <cloudformation 스택명> \
    --capabilities CAPABILITY_NAMED_IAM \
    --region <cloudformation을 배포 할 리전> \
    --profile <profile name>
```
SAM은 배포 시 
* 애플리케이션의 코드를 압축해서 S3에 업로드
* CodeUri가 S3 path로 변경된 CloudFormation 템플릿파일 생성, S3에 업로드
* S3에 업로드된 파일을 이용해서 Cloudformation 스택을 생성하여 배포

Cloudformation 콘솔에서 Template를 확인하시거나 S3에 업로드된 Template를 다운받아서 열어 보시면 Code Uri에 S3를 지정하고 있는 것을 확인 할 수 있습니다. 
```
CodeUri: s3://<zip파일을 업로드 할 Bucket name>/SAM/70b53b458e040c19f27aaf1d7f197e0e
```
Cloudformation 스택생성이 완료되면 Outputs에 람다함수의 새로 배포된 Version정보를 포함한 ARN정보를 표시해줍니다.

이 ARN을 Cloudfront의 리사이징용 Behaviors > Edge Function Associations > Function ARN/Name에 넣어주었습니다.


## 참고
* [https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction)
