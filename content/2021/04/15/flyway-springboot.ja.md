---
title: "Flyway 一部手動で作業する"
date: "2021-04-15"
tags: ["Flyway", "Spring"]
author: "bokyung"
---

Spring BootプロジェクトでFlywayを利用していますが、手動で作業しないといけないケースが出てきます。データが多いテーブルに手動でカラムを追加したり、indexを追加したりなどなど。
手動で作業した場合は誰がいつ作業したのかが管理できないため、履歴管理する方法がないかと思い、公式ドキュメントを読んでみました。
ignoreIgnoredMigrationsパラメーターを検証してみました。

## バージョン
* Flyway
  * flyway-core 7.1.1
* Spring Boot 2.4.1

## テーブル追加しサーバー起動
* sqlバージョンファイル追加
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
* サーバー起動
  * サーバー起動時、DBへ正常的にSQL文が反映されました。
    ```
    2021-04-15 11:06:14.196  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.029s)
    2021-04-15 11:06:14.209  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": << Empty Schema >>
    2021-04-15 11:06:14.218  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1.0.0 - Create accounts"
    2021-04-15 11:06:14.284  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version "1.1.0 - Modify accounts"
    2021-04-15 11:06:14.305  INFO 7888 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Successfully applied 2 migrations to schema "public" (execution time 00:00.104s)
    ```
  * flyway_schema_historyにV1.1.0まで反映されてういることが確認できます。
  {{< img src="/images/2021/0415/flyway_v1.png" alt="flyway_v1" link="/images/2021/0415/flyway_v1.png">}}
  * accountsテーブルにも変更されたカラムが反映されています。
  {{< img src="/images/2021/0415/user_v1.png" alt="user_v1" link="/images/2021/0415/user_v1.png">}}

## 手動でカラム変更しサーバー起動
* 手動でemailカラムを追加しサーバーを起動してみました。
  ```
  ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
  ```
* accountsテーブルにも変更されたカラムが反映されています。この状態でサーバーを起動します。
{{< img src="/images/2021/0415/user_v2.png" alt="user_v2" link="/images/2021/0415/user_v2.png">}}
* データマイグレーションなしにサーバーが起動できました。
  ```
  2021-04-15 11:02:38.735  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.031s)
  2021-04-15 11:02:38.749  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": 1.1.0
  2021-04-15 11:02:38.750  INFO 30080 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "public" is up to date. No migration necessary.
  ```

# 手動で作業したSQLをバージョン管理する
* sqlバージョンファイル追加
  * V1.0.9__Modify_accounts.sql
    ```
    ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
    ```
* この状態でサーバーを起動すると当然ながらマイグレーション失敗してしまいます。
  ```
  org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'flywayInitializer' defined in class path resource [org/springframework/boot/autoconfigure/flyway/FlywayAutoConfiguration$FlywayConfiguration.class]: Invocation of init method failed; nested exception is org.flywaydb.core.api.exception.FlywayValidateException: Validate failed: Migrations have failed validation
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1788) ~[spring-beans-5.3.2.jar:5.3.2]
  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:609) ~[spring-beans-5.3.2.jar:5.3.2]
  ```

# ignoreIgnoredMigrations パラメータを利用する
[Flyway 公式 Document](https://flywaydb.org/documentation/configuration/parameters/ignoreIgnoredMigrations)に色んなパラメータがありましたが、その中でignoreIgnoredMigrationsが目に入りました。
最新バージョンより前のバージョンファイルがあってもvalidate実行時、レポートされないと書かれています。
* application.propertiesに `ignore-ignored-migrations`オプションをtrueに設定します。
  ```
  spring.flyway.ignore-ignored-migrations=true
  ```
* sqlバージョンファイル追加
  * V1.0.9__Modify_accounts.sql
    ```
    ALTER TABLE accounts ADD COLUMN email VARCHAR ( 255 ) UNIQUE NOT NULL
    ```
* V1.0.9バージョンもカウントされ、データマイグレーションなしにサーバーが起動されました。
  ```
  2021-04-15 11:26:09.613  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 3 migrations (execution time 00:00.035s)
  2021-04-15 11:26:09.626  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema "public": 1.1.0
  2021-04-15 11:26:09.628  INFO 20004 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Schema "public" is up to date. No migration necessary.
  ```

これからバージョン番号にルールを付け、手動で作業したファイルも管理していきます。