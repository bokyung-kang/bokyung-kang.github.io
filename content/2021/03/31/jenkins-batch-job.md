---
title: "Jenkins에서 Spring Batch Job을 실행하기"
date: "2021-03-31"
tags: ["AWS", "Jenkins", "Spring"]
author: "bokyung"
---

Jenkins에서 스프링 배치 Job을 실행해 보았습니다.

## Jenkins 환경
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI 이름 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## 환경변수와 플러그인 설정
* 여러 Job을 등록하는 경우 각각의 Job에서 같은 파라미터를 넘겨주게 되는데 환경변수에 등록해두면 관리하기가 편해집니다.
* Job파라미터로 실행날짜를 넘겨주기 위해서 `timestamper`플러그인을 사용하였습니다.
{{< img src="/images/2021/0331/plugin-timestamper.png" alt="plugin-timestamper" link="/images/2021/0331/plugin-timestamper.png">}}

* `Manage Jenkins > System Configuration > Configure System > Global properties > Environment variables`에서 Job에서 공통으로 사용하는 환경변수를  설정합니다.
{{< img src="/images/2021/0331/global-env-setting.png" alt="global-env-setting" link="/images/2021/0331/global-env-setting.png">}}

## 배치Job등록
* `New Item > Freestyle project`을 선택해서 Job을 등록합니다. 
{{< img src="/images/2021/0331/job-build.png" alt="job-build" link="/images/2021/0331/job-build.png">}}

```
java -jar ${JAR_NAME} ${DB_HOST} ${DB_PORT} ${DB_NAME} ${DB_USER} ${DB_PASSWORD} \
--spring.batch.job.names=${JOB_NAME} version=${BUILD_NUMBER} requestDate=${BUILD_TIMESTAMP} --spring.batch.job.enabled=true --spring.profiles.active=dev
```

## 배치Job실행
* Job을 실행하고 실행 결과를 확인합니다.
```
2021-03-31 13:09:56.358  INFO 3564 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-03-31 13:09:56.379  INFO 3564 --- [           main] c.e.sample.batch.BatchApplicationKt      : Started BatchApplicationKt in 12.493 seconds (JVM running for 13.807)
2021-03-31 13:09:56.380  INFO 3564 --- [           main] o.s.b.a.b.JobLauncherApplicationRunner   : Running default command line with: [version=17, requestDate=2021-03-31]
2021-03-31 13:09:56.516  INFO 3564 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=jpaJob]] launched with the following parameters: [{requestDate=2021-03-31, version=17}]
2021-03-31 13:09:56.600  INFO 3564 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [jpaPagingItemStep]
2021-03-31 13:09:57.071  INFO 3564 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [jpaPagingItemStep] executed in 471ms
2021-03-31 13:09:57.092  INFO 3564 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=jpaJob]] completed with the following parameters: [{requestDate=2021-03-31, version=17}] and the following status: [COMPLETED] in 521ms
2021-03-31 13:09:57.127  INFO 3564 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
2021-03-31 13:09:57.129  INFO 3564 --- [           main] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2021-03-31 13:09:57.140  INFO 3564 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2021-03-31 13:09:57.155  INFO 3564 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
Finished: SUCCESS
```
## BatchApplication.kt
* Job실행 종료시 application도 종료시키기 위한 설정 
```
@EnableBatchProcessing
@SpringBootApplication
class BatchApplication

fun main(args: Array<String>) {
  val context = runApplication<BatchApplication>(*args)
  val exitCode = SpringApplication.exit(context)
  System.exit(exitCode)
}
```
