---
title: "Flyway 일부 수동으로 작업하기"
date: "2021-04-15"
tags: ["Flyway", "Spring"]
author: "bokyung"
---

스프링부트 프로젝트에서 Flyway를 사용하다 보면 수동으로 작업을 해야 하는 경우가 생깁니다.
데이터가 많은 테이블에 수동으로 컬럼이나 인덱스를 추가한다던가.
그런데 이렇게 되면 누가 수동으로 작업을 했는지 이력이 남지 않기때문에 이력관리도 할 수 있는 방법이 없을까하고 공식 Document를 찾아봤습니다.
ignoreIgnoredMigrations파라미터를 테스트해 보았습니다.

## 버전
* Flyway
  * flyway-core 7.1.1
* Spring Boot 2.4.1

## 테이블 추가, 서버 기동
* sql버전 파일 추가
  * V1.0.0__Create_accounts.sql
    ```
    CREATE TABLE accounts (
    user_id serial PRIMARY KEY,
    username VARCHAR ( 50 ) UNIQUE NOT NULL,
    password VARCHAR ( 50 ) NOT NULL,
    created_on TIMESTAMP NOT NULL
    );
    ```
  * V1.1.0__Modify_accounts.sql
    ```
    ALTER TABLE accounts ADD COLUMN last_login TIMESTAMP;
    ```
* 서버기동
  * 서버 기동 시 DB에 정상적으로 SQL문이 반영되었습니다.
    ```
    2021-04-15 11:06:14.196  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.029s)
    2021-04-15 11:06:14.209  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": << Empty Schema >>
    2021-04-15 11:06:14.218  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1.0.0 - Create accounts"
    2021-04-15 11:06:14.284  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1.1.0 - Modify accounts"
    2021-04-15 11:06:14.305  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Successfully applied 2 migrations to schema "public" (execution time 00:00.104s)
    ```
  * flyway_schema_history에 V1.1.0까지 잘 반영되어있는것을 확인할 수 있습니다.
  {{< img src="/images/2021/0415/flyway_v1.png" alt="flyway_v1" link="/images/2021/0415/flyway_v1.png">}}
  * accounts테이블에도 변경된 컬럼이 반영되어 있습니다.
  {{< img src="/images/2021/0415/user_v1.png" alt="user_v1" link="/images/2021/0415/user_v1.png">}}

## 수동으로 컬럼 변경, 서버 기동
* 수동으로 email컬럼을 추가 후에 서버를 기동해보겠습니다.
  ```
  ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
  ```
* accounts테이블에도 변경된 컬럼이 반영되어 있습니다. 이 상태로 서버를 기동해보겠습니다.
{{< img src="/images/2021/0415/user_v2.png" alt="user_v2" link="/images/2021/0415/user_v2.png">}}
* 데이터 마이그레이션없이 서버가 잘 기동되었네요.
  ```
  2021-04-15 11:02:38.735  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.031s)
  2021-04-15 11:02:38.749  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": 1.1.0
  2021-04-15 11:02:38.750  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "public" is up to date. No migration necessary.
  ```

# 수동작업 한 SQL을 버전관리하기
* sql버전 파일 추가
  * V1.0.9__Modify_accounts.sql
    ```
    ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
    ```
* 이 상태로 서버를 실행하면 당연히 마이그레이션 실패를 합니다.
  ```
  org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'flywayInitializer' defined in class path resource [org/springframework/boot/autoconfigure/flyway/FlywayAutoConfiguration$FlywayConfiguration.class]: Invocation of init method failed; nested exception is org.flywaydb.core.api.exception.FlywayValidateException: Validate failed: Migrations have failed validation
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1788) ~[spring-beans-5.3.2.jar:5.3.2]
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:609) ~[spring-beans-5.3.2.jar:5.3.2]
  ```

# ignoreIgnoredMigrations 파라미터 이용하기
[공식 Document](https://flywaydb.org/documentation/configuration/parameters/ignoreIgnoredMigrations)에 여러 파라미터가 있는데 그중에 ignoreIgnoredMigrations파라미터가 눈에 들어왔습니다.
최신 버전보다 이전 버전의 파일이 있어도 validate실행시 레포트 되지 않는다고 나와 있네요.
* application.properties에 `ignore-ignored-migrations`옵션을 true로 설정해줍니다.
  ```
  spring.flyway.ignore-ignored-migrations=true
  ```
* sql버전 파일 추가
  * V1.0.9__Modify_accounts.sql
    ```
    ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
    ```
* V1.0.9버전도 카운트되어 마이그레이션없이 서버가 잘 기동되었네요.
  ```
  2021-04-15 11:26:09.613  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 3 migrations (execution time 00:00.035s)
  2021-04-15 11:26:09.626  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": 1.1.0
  2021-04-15 11:26:09.628  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "public" is up to date. No migration necessary.
  ```

앞으로 버전번호에 룰을 정해서 수동적용 한 파일들은 관리해야겠습니다.