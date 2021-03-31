---
title: "Jenkins環境でSpring Batch Jobを実行する"
date: "2021-03-31"
tags: ["AWS", "Jenkins", "Spring"]
author: "bokyung"
---

JenkinsでSpring Batch Jobを実行してみました。

## Jenkinsの環境
* Amazon Linux2
  * AMI ID : ami-0f27d081df46f326c
  * AMI名 : amzn2-ami-hvm-2.0.20210303.0-x86_64-gp2
* Jenkins
  * version : Jenkins 2.284

## 環境変数とプラグイン設定
* 複数のJobを登録する場合は、各Jobに同じパラメータを設定することになりますが、環境変数に登録しておくと一括管理ができます。
* Jobパラメーターに実行日を渡すために、`timestamper`プラグインを利用しました。
{{< img src="/images/2021/0331/plugin-timestamper.png" alt="plugin-timestamper" link="/images/2021/0331/plugin-timestamper.png">}}

* `Manage Jenkins > System Configuration > Configure System > Global properties > Environment variables`でJob共通で利用している環境変数を設定します。 
{{< img src="/images/2021/0331/global-env-setting.png" alt="global-env-setting" link="/images/2021/0331/global-env-setting.png">}}

## Batch Job登録
* `New Item > Freestyle project`を選び、JOBを登録します。
{{< img src="/images/2021/0331/job-build.png" alt="job-build" link="/images/2021/0331/job-build.png">}}
```
java -jar ${JAR_NAME} ${DB_HOST} ${DB_PORT} ${DB_NAME} ${DB_USER} ${DB_PASSWORD} \
--spring.batch.job.names=${JOB_NAME} version=${BUILD_NUMBER} requestDate=${BUILD_TIMESTAMP} --spring.batch.job.enabled=true --spring.profiles.active=dev
```

## Batch Job実行
* Jobを実行し、実行結果を確認します。
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
* Job実行終了し、applicationも終了させるための設定。 
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
