---
title: "CircleCI의 Contexts를 이용하여 AWS ECS 자동 배포하기"
date: "2021-02-02"
tags: ["circleci", "AWS", "ECS"]
author: "bokyung"
---

springboot + kotlin으로 개발한 backendAPI를 CircleCI로 배포하고 있습니다.
staging/production 환경별 환경변수를 Contexts를 이용하면 좀 더 편리하게 설정할 수 있습니다.

## Contexts 설정
Organization Settings > Contexts 에서 서비스의 각 환경별 환경변수를 설정합니다.
staging/production 환경 이름으로 등록했습니다.
{{< img src="/images/2021/0202/contexts.png" alt="contexts" link="/images/2021/0202/contexts.png">}}

staging 환경의 환경변수입니다.
ecr와 ecs용 orbs를 이용하기 위해서 AWS 환경변수를 추가하고 프로젝트에서 공통으로 사용하기 위해서 SERVICE_PREFIX를 추가했습니다.
{{< img src="/images/2021/0202/contexts_staging.png" alt="contexts_staging" link="/images/2021/0202/contexts_staging.png">}}

## CircleCI config.yml 설정               
### build-and-push-image
* backend-build를 실행합니다.
* release 브랜치인 경우 build-and-push-image-staging을 실행합니다.
* master 브랜치인 경우 build-and-push-image-production을 실행합니다.
* context에는 CircleCI Contexts에서 지정한 이름을 추가합니다.
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
* backend-build를 실행합니다.
* release 브랜치인 경우 build-and-push-image-staging의 실행이 완료되면 deploy-service-update-staging을 실행합니다.
* master 브랜치인 경우 build-and-push-image-production의 실행이 완료되면 build-and-push-image-production을 실행합니다.
* context에는 CircleCI Contexts에서 지정한 이름을 추가합니다.

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

## CircleCI 배포
### release/20210202_1 브랜치를 push
{{< img src="/images/2021/0202/circleci_build.png" alt="circleci_build" link="/images/2021/0202/circleci_build.png">}}

### 새 작업정의가 등록된것을 확인 `task-definition/*****-backend:5`
{{< img src="/images/2021/0202/circleci_service_update.png" alt="circleci_service_update" link="/images/2021/0202/circleci_service_update.png">}}

### AWS ECS 확인
* 작업정의 `demo1-backend:5`로 작업이 기동된 것을 확인할 수 있습니다.
{{< img src="/images/2021/0202/aws_ecs.png" alt="aws_ecs" link="/images/2021/0202/aws_ecs.png">}}