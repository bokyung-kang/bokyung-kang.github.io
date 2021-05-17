---
title: "AWS SAMを利用して、Lambda関数デプロイする"
date: "2021-05-17"
tags: ["AWS", "AWS SAM", "AWS Lambda", "AWS S3"]
author: "bokyung"
---

前のPostに[CloudFrontとLambda@Edgeを利用して画像をリサイズさせる]({{< ref "/2021/05/14/lambda-edge-resize.ja" >}})方法に関する内容がありましたが、AWS SAMを利用してCloudFormationでLambda関数をデプロイする方法をまとめてみました。

## デプロイ順番
### SAM CLIインストール
[Install SAM CLI](https://aws.amazon.com/ko/serverless/SAM/)

私は `SAM CLI, version 1.23.0`を利用しました。
### template.yaml作成
CloudFormation基盤のテンプレート(template.yaml)を作成します。
```shell
.
├── index.js
├── node_modules/
├── template.yaml
├── package-lock.json
└── package.json
```
{{< gist bokyung-kang aa6452d3d4f559c4d77ac11fa5bdc876 >}}

### SAM deployを利用した、コードのパッケージングとデプロイ

```shell
SAM deploy \
    --template-file template.yaml \
    # Lambda@edgeの場合はus-east-1リージョンに作成したBucketが必要です。
    --s3-bucket <zipファイルをアップロードするBucket name> \
    --s3-prefix SAM \
    --stack-name <cloudformation スタック名> \
    --capabilities CAPABILITY_NAMED_IAM \
    --region <cloudformationをデプロイするリージョン名> \
    --profile <profile name>
```
SAMはデプロイ時、
* アプリケーションのコードを圧縮して、S3へアップロード
* CodeUriがS3 pathに変換されたCloudFormationテンプレートファイル作成、S3へアップロード
* S3にアップロードされたファイルを利用して、Cloudformationのスタック作成してデプロイ 

Cloudformationコンソール上でTemplateの中身を確認するとCodeUriがS3のパスになっていることが確認できます。

```
CodeUri: s3://<zipファイルをアップロードするBucket name>/SAM/70b53b458e040c19f27aaf1d7f197e0e
```

Cloudformationのスタック作成が完了されたら、OutputsにLambda関数の新たなバージョンを含めたARNを表示してくれます。

このARNをCloudfrontのリサイズ用のBehaviors > Edge Function Associations > Function ARN/Nameへ指定します。


## 参考
* https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction)
