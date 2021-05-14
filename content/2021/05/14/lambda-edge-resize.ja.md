---
title: "CloudFrontとLambda@Edgeを利用して画像をリサイズさせる。"
date: "2021-05-14"
tags: ["AWS", "CloudFront", "Lambda@Edge"]
author: "bokyung"
---

リサイズされた画像をS3へ保存せずに、原本のみ利用してリサイズさせる方法がないか調べてみましたが、Lambda@Edgeを利用してリサイズさせる方法があったので、試してみました。

## Lambda@Edge動作概要
{{< img src="/images/2021/0514/lambda-edge.png" alt="lambda-edge" link="/images/2021/0514/lambda-edge.png">}}
Lambda@EdgeはCloudFrontへアクセスする時に実行されるLambdaの拡張版です。
CloudFrontイベント発生時、Lambda関数の実行が可能です。
イベントには４種類があります。
* Viewer Request : CloudFrontがビューアーからのリクエストを受け、リクエストしたオブジェクトがedge cacheにあるかを確認する前に関数を実行します。
* Origin Request : CloudFrontがオリジンにリクエストを渡す場合のみ実行されます。 リクエストしたオブジェクトがedge cacheにある場合は関数は実行されません。
* Origin Response : CloudFrontがオリジンからレスポンスを受け取った後、レスポンスへオブジェクトをcacheする前に関数を実行します。
* Viewer Response : リクエストしたオブジェクトをビューアーに返却する前に実行されます。 この関数は、オブジェクトがedge cacheに既に存在しているか否かに関わらず実行されます。

## Lambda@Edge 注意点
* 環境変数は利用不可
* us-east-1 リージョンのみ対応
* イベントタイプによって異なるクォータ
  | Entity | Origin request and response event quotas | Viewer request and response event quotas |
  | -------- | -------- | -------- |
  | Function memory size    |Same as Lambda quotas（128 MB to 10,240 MB）   | 128 MB   |
  | Function timeout|  30 seconds|5 seconds  |
  | Size of a response  |1 MB  |  40 KB|
  | Maximum compressed size of a Lambda function and any included libraries |  50 MB|1 MB  |

## イメージリサイジング処理実装
### イメージリサイジング処理の流れ
Origin Responseイベント発生時、Lambda@Edge関数を実行する方法を利用しました。
* https://images.example.com/images/heic_image.heic?w=1500&h=1500 へアクセス
* CloudFrontから画像データをリクエストしたが、cacheされてない状態。
* CloudFrontがS3オリジンへリクエスト。
* S3オリジンに画像が存在したら、S3オリジンから応答。
* Lambda関数を実行し、画像をリサイジング。
* リサイジング後の画像をCloudFrontへcache処理するようにリクエスト。
* CloudFrontはcache処理後、ブラウザーにリサイズされた画像を表示。

### CloudFront 設定
CloudFormationを利用して構築しました。DistributionConfigの設定の中で一部を見てみましょう。
{{< gist bokyung-kang e0cc098d446698288849019e42046e7c >}}

### Lambda@Edge IAM Role作成
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
### Lambda関数作成

Sharpリサイズ用のパッケージインストール `npm install --arch=x64 --platform=linux sharp`
[AWS Lambda用 Sharpインストールガイド](https://sharp.pixelplumbing.com/install#aws-lambda)

#### index.js作成
{{< gist bokyung-kang ddfb1a8d3fd66cdf674ff553c47457d1 >}}

#### zipファイルアップロード
index.js, node_modulesをzipファイルに圧縮し、Code source > Upload from .zip fileを選択して、ファイルをアップロードします。
RuntimeはNode.js 14.xを選びます。

```shell
.
├── index.js
├── node_modules/
└── package.json
```
#### Lambda@Edgeデプロイ
Actions > Deploy to Lambda@Edgeをクリックし、Lambda@Edgeをデプロイします。
{{< img src="/images/2021/0514/cloudfront_lambda_deploy.png" alt="cloudfront_lambda_deploy" link="/images/2021/0514/cloudfront_lambda_deploy.png">}}

### リサイズテスト
#### 画像リサイズ
heicをjpegへ変換して、リサイズされていることが確認できます。
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

再度アクセスすると、cacheが適用されていることが確認できます。
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
#### Cloudfront SignedURLを利用する場合
* SignedURLを発行します。
```shell
aws cloudfront sign \
    --url https://images.example.com/images/heic_image.heic?w=1400&h=1400 \
    --key-pair-id K2XRXXXXXXXXXX \
    --private-key file:///<key path>/private_key.pem \
    --date-less-than 2021-05-14T10:50:00+09:00 \
    --profile <profile name>

https://images.example.com/images/heic_image.heic?w=1400&h=1400&Expires=1620957000&Signature=NLDdoaOrD0S8hyEIBzH6Q0otJagaRmUUtX3ZY0uuA0QIzc1e5D6z-ddQ~k0U1~4WQSiK-sDGplt-CPDiUTjz053yFlaDWzQohNybLLVRcUaKRXDgl~ahJPwjfstnKzzOl4g3Z628ZD-8UkLPnPaUx~Ibeo2Z55GFp8Ih0aNBZuPdd0al~4X5~lUGrsgZvRfg0QWis1X3VvShWPLL3nIphAFtJJu0~IfzyKNRZMQMTBvJcqL-ifdA2uj99VHGz-pec7r33y38TW5sirS6kQVJHe9WmAxjDQhTO42M04oSHwu~t6Mrxh3~DalyxEM0wcQ5yWOAKD9FA4~A7im-9tbBTg__&Key-Pair-Id=K2XRXXXXXXXXXX
```
* SignedURLへアクセスします。heicをjpegへ変換して、リサイズされていることが確認できます。
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
## 参考
* [https://sharp.pixelplumbing.com/install#aws-lambda](https://sharp.pixelplumbing.com/install#aws-lambda)
* [https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html)
* [https://aws.amazon.com/blogs/networking-and-content-delivery/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/](https://aws.amazon.com/blogs/networking-and-content-delivery/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/)