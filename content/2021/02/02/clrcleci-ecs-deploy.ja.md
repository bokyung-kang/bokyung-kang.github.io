---
title: "CircleCIのContextsを利用し、AWS ECSへデプロイする"
date: "2021-02-02"
tags: ["circleci", "AWS", "ECS"]
author: "bokyung"
---

springboot + kotlinで開発したbackendAPIをCircleCIを利用してデプロイしています。
staging/production環境毎の環境変数はContextsを利用すればもっと簡単に設定することが可能です。

## Contexts設定
Organization Settings > Contexts でサービスの各環境毎の環境変数を設定します。
staging/productionの環境名で追加しました。
![contexts](/images/2021/0202/contexts.png)

staging環境の環境変数です。
ecrとecsのorbsを利用するため、AWSの環境変数を追加し、プロジェクト内で共通で利用するためにSERVICE_PREFIXを追加しました。
![contexts_staging](/images/2021/0202/contexts_staging.png)

## CircleCI config.yml 設定               
### build-and-push-image
* backend-buildを実行します。
* releaseブランチの場合、build-and-push-image-stagingを実行します。
* masterブランチの場合、build-and-push-image-productionを実行します。
* contextにはCircleCI Contextsで指定した名前を指定します。
```yml
version: 2.1
orbs:
  aws-ecr: circleci/aws-ecr@6.15.3
  aws-ecs: circleci/aws-ecs@1.4.0
  gradle: circleci/gradle@2.2.0

executors:
  openjdk-executor:
    docker:
      - image: circleci/openjdk:14-jdk-buster-node-browsers
        .....
jobs:
  backend-build:
    executor: openjdk-executor
    .....

workflows:
  build-deploy:
    jobs:
      - backend-build
      - aws-ecr/build-and-push-image:
          name: build-and-push-image-staging
          requires:
            - backend-build
          context: DEMO_STAGING
          attach-workspace: true
          checkout: false
          repo: '${SERVICE_PREFIX}'
          tag: 'latest,${CIRCLE_SHA1}'
          filters:
            branches:
              only:
                - /release\/.*/

      - aws-ecr/build-and-push-image:
          name: build-and-push-image-production
          requires:
            - backend-build
          context: DEMO_PRODUCTION
          attach-workspace: true
          checkout: false
          repo: '${SERVICE_PREFIX}'
          tag: 'latest,${CIRCLE_SHA1}'
          filters:
            branches:
              only:
                - master
 
```

### deploy-service-update
* backend-buildを実行します。
* releaseブランチの場合、build-and-push-image-stagingの実行が完了されたらdeploy-service-update-stagingを実行します。
* masteブランチの場合、build-and-push-image-productionの実行が完了されたらbuild-and-push-image-productionを実行します。
* contextにはCircleCI Contextsで指定した名前を指定します。

```yml
- aws-ecs/deploy-service-update:
    name: deploy-service-update-staging
    requires:
      - build-and-push-image-staging
    context: DEMO_STAGING
    family: '${SERVICE_PREFIX}-backend'
    cluster-name: '${SERVICE_PREFIX}-backend'
    service-name: '${SERVICE_PREFIX}-backend-ec2-service'
    container-image-name-updates: 'container=${SERVICE_PREFIX}-backend,tag=${CIRCLE_SHA1}'
    filters:
      branches:
        only:
          - /release\/.*/

- aws-ecs/deploy-service-update:
    name: deploy-service-update-production
    requires:
      - build-and-push-image-production
    context: DEMO_PRODUCTION
    family: '${SERVICE_PREFIX}-backend'
    cluster-name: '${SERVICE_PREFIX}-backend'
    service-name: '${SERVICE_PREFIX}-backend-ec2-service'
    container-image-name-updates: 'container=${SERVICE_PREFIX}-backend,tag=${CIRCLE_SHA1}'
    filters:
      branches:
        only:
          - master
```

## CircleCI デプロイ
### release/20210202_1 ブランチを push
![circleci_build](/images/2021/0202/circleci_build.png)

### 新たなタスク定義が登録されていることを確認 `task-definition/*****-backend:5`
![circleci_service_update](/images/2021/0202/circleci_service_update.png)

### AWS ECS 確認
* タスク定義 `demo1-backend:5`でタスクが起動されていることが確認できます。
![aws_ecs](/images/2021/0202/aws_ecs.png)