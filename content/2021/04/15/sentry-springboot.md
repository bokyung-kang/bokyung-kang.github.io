---
title: "Sentry를 스프링부트 프로젝트에 적용하기"
date: "2021-04-15"
tags: ["Sentry", "Spring"]
author: "bokyung"
---

react.js에 Sentry를 적용하려고 찾아보던 중에 스프링부트도 대응언어에 포함되어있어서 테스트해 보았습니다.

## 버전
* Spring Boot 2.4.1
* Sentry  
  * sentry-spring-boot-starter 4.3.0

## Sentry에 프로젝트 추가 및 설정
* Sentry에 로그인 후 프로젝트를 생성합니다.
* Client Keys (DSN)메뉴에 있는 DSN키를 복사해둡니다.
  {{< img src="/images/2021/0415/spring-dsm.png" alt="spring-dsm" link="/images/2021/0415/spring-dsm.png">}}
* build.gradle.kts에 의존관계를 추가합니다.
  * build.gradle.kts
    ```
    implementation("io.sentry:sentry-spring-boot-starter:4.3.0")
    ```
* application.properties에 DSN (Data Source Name)을 설정합니다.
  * application.properties
    ```
    # DSN설정
    sentry.dsn=https://xxxxxxxxxxxxxxxxxxxxxxxxxxx@xxxxxx.ingest.sentry.io/xxxxx
    # 에러 추척 설정
    sentry.enable-tracing=true
    ```
  * application-development.properties
    ```
    # 각각의 환경도 설정할 수 있습니다. 
    sentry.environment=development
    ```

* Sentry에 에러를 보내기 위한 설정
  * RestApi프로젝트라 `@RestControllerAdvice`을 이용하여 공통 예외 처리를 추가했습니다.
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

## 에러를 발생시켜 Sentry상세페이지 확인
* Content type을 추가해서 에러를 발생시켜보겠습니다.
{{< img src="/images/2021/0415/springboot-error.png" alt="springboot-error" link="/images/2021/0415/springboot-error.png">}}
* Search by Trace를 클릭하면 에러 추척도 가능하네요.
{{< img src="/images/2021/0415/springboot-error-trace.png" alt="springboot-error-trace" link="/images/2021/0415/springboot-error-trace.png">}}
* Dashboard도 제공하네요.　Custom Dashboards는 Business플랜부터 가능하네요.
{{< img src="/images/2021/0415/springboot-sentry.png" alt="springboot-sentry" link="/images/2021/0415/springboot-sentry.png">}}

## slack에 통지
* slack이나 타 서비스와 연계하려면 `Team`플랜부터 가능하네요.
{{< img src="/images/2021/0415/sentry-slack.png" alt="sentry-slack" link="/images/2021/0415/sentry-slack.png">}}

Business플랜이면 Amazon SQS에 데이터 전송도 가능하다니 편리할 것 같습니다.
나중에 [공식사이트](https://docs.sentry.io/platforms/java/guides/spring-boot/)를 꼼꼼히 살펴보아야겠습니다.