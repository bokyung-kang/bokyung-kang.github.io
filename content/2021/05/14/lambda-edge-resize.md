---
title: "CloudFront와 Lambda@Edge를 이용해서 이미지 리사이징하기"
date: "2021-05-14"
tags: ["AWS", "AWS CloudFront", "AWS Lambda@Edge", "Image Resizing", "AWS S3"]
author: "bokyung"
---

리사이징 된 이미지를 S3에 저장하지 않고 원본만 이용해서 리사이징 하는 방법은 없을까 고민하던차에 Lambda@Edge를 이용해서 이미지 리사이징 하는 방법을 적용해 보았습니다.

## Lambda@Edge 동작 원리
{{< img src="/images/2021/0514/lambda-edge.png" alt="lambda-edge" link="/images/2021/0514/lambda-edge.png">}}
Lambda@Edge는 CloudFront에 접근할 때 실행되는 Lambda의 확장판입니다.
CloudFront 이벤트가 발생할 때 Lambda 함수를 실행할 수 있습니다.
이벤트는 4가지가 있습니다.
* Viewer Request : CloudFront가 뷰어로부터 요청을 받고 요청한 개체가 edge cache에 있는지 확인하기 전에 함수를 실행합니다.
* Origin Request : CloudFront가 오리진으로 요청을 전달할 때만 실행됩니다. 요청한 개체가 edge cache에 있으면 함수가 실행되지 않습니다.
* Origin Response : CloudFront가 오리진으로부터 응답을 받은 후 응답에 개체를 cache하기 전에 함수를 실행합니다.
* Viewer Response : 요청한 개체를 뷰어에 반환하기 전에 기능이 실행됩니다. 이 함수는 개체가 edge cache에 이미 있는지 여부에 관계없이 실행됩니다.

## Lambda@Edge 주의점
* 환경변수는 사용 불가
* us-east-1 리전만 대응
* 이벤트 유형에 따라 다른 할당량
  | Entity | Origin request and response event quotas | Viewer request and response event quotas |
  | -------- | -------- | -------- |
  | Function memory size    |Same as Lambda quotas（128 MB to 10,240 MB）   | 128 MB   |
  | Function timeout|  30 seconds|5 seconds  |
  | Size of a response  |1 MB  |  40 KB|
  | Maximum compressed size of a Lambda function and any included libraries |  50 MB|1 MB  |

## 이미지 리사이징 구현
### 이미지 리사이징 처리 흐름
Origin Response이벤트 발생 시, Lambda@Edge함수 실행하는 방법을 이용했습니다.
* https://images.example.com/images/heic_image.heic?w=1500&h=1500 에 접속
* CloudFront에서 이미지를 요청했지만, cache되어있지 않은 상태.
* CloudFront가 S3오리진에 요청.
* S3오리진에 이미지가 존재하면, S3오리진에서 응답.
* Lambda 함수를 실행하여 이미지 리사이징.
* 리사이징 한 이미지를 CloudFront에 캐싱처리 요청.
* CloudFront는 캐싱 후, 브라우저에 리사이징 된 이미지를 표시.

### CloudFront 설정
CloudFormation을 이용해서 구축했습니다. DistributionConfig 설정 중 일부를 살펴보겠습니다.
{{< gist bokyung-kang e1b8475c7b0b08c7e0c9dea8276c712e >}}

### Lambda@Edge IAM Role 작성
#### Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "cloudfront:UpdateDistribution",
        "iam:CreateServiceLinkedRole",
        "s3:GetObject",
        "lambda:EnableReplication",
        "lambda:GetFunction",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```
#### Trust Relationship Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "edgelambda.amazonaws.com",
          "lambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
### Lambda 함수 작성

Sharp 리사이징용 패키지 설치 `npm install --arch=x64 --platform=linux sharp`
[AWS Lambda용 Sharp 설치 가이드](https://sharp.pixelplumbing.com/install#aws-lambda)
#### index.js 작성
{{< gist bokyung-kang a956553c09e6cf737e3caeead15d5a81 >}}

#### zip파일 업로드
index.js, node_modules을 zip파일로 압축하고 Code source > Upload from .zip file을 선택해서 파일을 업로드합니다.
Runtime은 Node.js 14.x를 선택합니다.

```shell
.
├── index.js
├── node_modules/
└── package.json
```
#### Lambda@Edge 배포
Actions > Deploy to Lambda@Edge를 클릭하고 Lambda@Edge를 배포합니다.
{{< img src="/images/2021/0514/cloudfront_lambda_deploy.png" alt="cloudfront_lambda_deploy" link="/images/2021/0514/cloudfront_lambda_deploy.png">}}

### 리사이징 테스트
#### 이미지 리사이징
heic파일을 jpeg로 변경하여 리사이징에 성공한 것을 확인할 수 있습니다.
```shell
curl -I 'https://images.example.com/images/heic_image.heic?w=1400&h=1400'
HTTP/2 200
content-type: image/jpeg
content-length: 356013
date: Fri, 14 May 2021 01:50:28 GMT
last-modified: Fri, 14 May 2021 01:37:54 GMT
etag: "9835f7aa1198532d0b538axxxxxxxxxx"
accept-ranges: bytes
server: AmazonS3
x-cache: Miss from cloudfront
via: 1.1 186a60433f9963bxxxxxxxxxxxxxx.cloudfront.net (CloudFront)
x-amz-cf-pop: NRT20-C2
x-amz-cf-id: dihScCemgN8p_VVykjhmXZhkaRP1t7B_y0o_WZoJMK17hexhbHMU0g==
```

다시 접속하면 cache가 적용된 것을 확인할 수 있습니다.
```
curl -I 'https://images.example.com/images/heic_image.heic?w=1400&h=1400'
HTTP/2 200
content-type: image/jpeg
content-length: 356013
date: Fri, 14 May 2021 01:50:28 GMT
last-modified: Fri, 14 May 2021 01:37:54 GMT
etag: "9835f7aa1198532d0b538axxxxxxxxxx"
accept-ranges: bytes
server: AmazonS3
x-cache: Hit from cloudfront
via: 1.1 65866bb6c20ad09xxxxxxxxxxxxxxx.cloudfront.net (CloudFront)
x-amz-cf-pop: NRT57-C2
x-amz-cf-id: C_g2WMkGFgKPtOX87AZBI6J66WDBDARs0pN23DBJSdMam6mUHntFEA==
age: 3
```
#### Cloudfront SignedURL을 사용하는 경우
* SignedURL을 발행합니다.
```shell
aws cloudfront sign \
    --url https://images.example.com/images/heic_image.heic?w=1400&h=1400 \
    --key-pair-id K2XRXXXXXXXXXX \
    --private-key file:///<key path>/private_key.pem \
    --date-less-than 2021-05-14T10:50:00+09:00 \
    --profile <profile name>

https://images.example.com/images/heic_image.heic?w=1400&h=1400&Expires=1620957000&Signature=NLDdoaOrD0S8hyEIBzH6Q0otJagaRmUUtX3ZY0uuA0QIzc1e5D6z-ddQ~k0U1~4WQSiK-sDGplt-CPDiUTjz053yFlaDWzQohNybLLVRcUaKRXDgl~ahJPwjfstnKzzOl4g3Z628ZD-8UkLPnPaUx~Ibeo2Z55GFp8Ih0aNBZuPdd0al~4X5~lUGrsgZvRfg0QWis1X3VvShWPLL3nIphAFtJJu0~IfzyKNRZMQMTBvJcqL-ifdA2uj99VHGz-pec7r33y38TW5sirS6kQVJHe9WmAxjDQhTO42M04oSHwu~t6Mrxh3~DalyxEM0wcQ5yWOAKD9FA4~A7im-9tbBTg__&Key-Pair-Id=K2XRXXXXXXXXXX
```
* SignedURL에 접속 해 봅니다. heic파일을 jpeg로 변경하여 리사이징에 성공한 것을 확인할 수 있습니다.
```shell
curl -I 'https://images.example.com/images/heic_image.heic?w=1400&h=1400&Expires=1620957000&Signature=NLDdoaOrD0S8hyEIBzH6Q0otJagaRmUUtX3ZY0uuA0QIzc1e5D6z-ddQ~k0U1~4WQSiK-sDGplt-CPDiUTjz053yFlaDWzQohNybLLVRcUaKRXDgl~ahJPwjfstnKzzOl4g3Z628ZD-8UkLPnPaUx~Ibeo2Z55GFp8Ih0aNBZuPdd0al~4X5~lUGrsgZvRfg0QWis1X3VvShWPLL3nIphAFtJJu0~IfzyKNRZMQMTBvJcqL-ifdA2uj99VHGz-pec7r33y38TW5sirS6kQVJHe9WmAxjDQhTO42M04oSHwu~t6Mrxh3~DalyxEM0wcQ5yWOAKD9FA4~A7im-9tbBTg__&Key-Pair-Id=K2XRXXXXXXXXXX'

HTTP/2 200
content-type: image/jpeg
content-length: 356013
date: Fri, 14 May 2021 01:45:27 GMT
last-modified: Fri, 14 May 2021 01:37:54 GMT
etag: "9835f7aa1198532d0b538axxxxxxxxxx"
accept-ranges: bytes
server: AmazonS3
x-cache: Miss from cloudfront
via: 1.1 25d5704e1dc4bae769b7dexxxxxxxxxx.cloudfront.net (CloudFront)
x-amz-cf-pop: NRT57-C2
x-amz-cf-id: 1pJUhQFKbilToyrRCp4_IwkBec1ki3fh5G-Y0fltGoTeRA3WUsymPQ==

```
## 참고
* [https://sharp.pixelplumbing.com/install#aws-lambda](https://sharp.pixelplumbing.com/install#aws-lambda)
* [https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html)
* [https://aws.amazon.com/blogs/networking-and-content-delivery/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/](https://aws.amazon.com/blogs/networking-and-content-delivery/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/)