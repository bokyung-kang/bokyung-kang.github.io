---
title: "SentryをSpringBootプロジェクトへ適用する"
date: "2021-04-15"
tags: ["Sentry", "Spring"]
author: "bokyung"
---

react.jsへSentryを適用しようと思い、公式ドキュメントを見てみると対応言語の中にSpringBootがあったため、試してみました。

## バージョン
* Spring Boot 2.4.1
* Sentry  
  * sentry-spring-boot-starter 4.3.0

## Sentryへプロジェクト追加及び設定
* Sentryへログイン後、プロジェクト作成。
* Client Keys (DSN)メニューに書いてある、DSNキーをコピーしておきます。
  {{< img src="/images/2021/0415/spring-dsm.png" alt="spring-dsm" link="/images/2021/0415/spring-dsm.png">}}
* build.gradle.ktsに依存関係を追加します。
  * build.gradle.kts
    ```
    implementation("io.sentry:sentry-spring-boot-starter:4.3.0")
    ```
* application.properties에 DSN (Data Source Name)を設定します。
  * application.properties
    ```
    # DSN設定
    sentry.dsn=https://xxxxxxxxxxxxxxxxxxxxxxxxxxx@xxxxxx.ingest.sentry.io/xxxxx
    # エラートレース設定
    sentry.enable-tracing=true
    ```
  * application-development.properties
    ```
    # 各環境の設定も可能です。
    sentry.environment=development
    ```

* Sentryにエラーを送るための設定
  * サンプルプロジェクトがRestApiプロジェクトなので、`@RestControllerAdvice`を利用して共通エラー処理を追加しました。
    ```
    @RestControllerAdvice
    class SampleControllerAdvice {

        @ResponseStatus(HttpStatus.BAD_REQUEST)
        @ExceptionHandler(MissingPathVariableException::class)
        fun handleMissingPathVariable(ex: MissingPathVariableException): Map<String, String> {
            Sentry.captureException(ex)
            val error: Map<String, String> = mapOf("code" to "E0001", "message" to "Parameter error")
            return error
        }

        @ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
        @ExceptionHandler(HttpMediaTypeNotSupportedException::class)
        fun handleHttpMediaTypeNotSupported(ex: HttpMediaTypeNotSupportedException): Map<String, String> {
            Sentry.captureException(ex)
            val error: Map<String, String> = mapOf("code" to "E0002", "message" to "Unsupported Media Type")
            return error
        }

        @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
        @ExceptionHandler(Exception::class)
        fun handleExceptions(ex: Exception): Map<String, String> {
            Sentry.captureException(ex)
            val error: Map<String, String> = mapOf("code" to "E0003", "message" to "Internal Server Error")
            return error
        }
    }
    ```

## エラーを発生させ、Sentry側を確認
* Content typeを追加し、エラーを発生させてみました。
{{< img src="/images/2021/0415/springboot-error.png" alt="springboot-error" link="/images/2021/0415/springboot-error.png">}}
* Search by Traceをクリックするとエラートレースも可能です。
{{< img src="/images/2021/0415/springboot-error-trace.png" alt="springboot-error-trace" link="/images/2021/0415/springboot-error-trace.png">}}
* Dashboardも提供しています。カスタムDashboardはBusinessプランから使えるそうです。
{{< img src="/images/2021/0415/springboot-sentry.png" alt="springboot-sentry" link="/images/2021/0415/springboot-sentry.png">}}

## slackへ通知
* slackとか他のサービスと連携するためには、`Team`プランにする必要があります。slackへのメッセージ通知のみならsentryの`WebHooks`を利用すれば`Developer(無料)`ブランでも利用可能です。
{{< img src="/images/2021/0415/sentry-slack.png" alt="sentry-slack" link="/images/2021/0415/sentry-slack.png">}}

Businessプランの場合、Amazon SQSへデータ転送もできると書いてあるので、便利そうです。
後で [公式サイド](https://docs.sentry.io/platforms/java/guides/spring-boot/)をよく見てみます。