---
title: "Sentry를 리액트 프로젝트에 적용하기"
date: "2021-04-16"
tags: ["Sentry", "React.js"]
author: "bokyung"
---

 [공식사이트](https://docs.sentry.io/platforms/javascript/guides/react/)를 보면서 react.js에 Sentry를 적용해 보았습니다.

## 버전
* react 17.0.1
* Sentry
  * @sentry/react 6.2.5
  * @sentry/tracing 6.2.5 

{{< adsense >}}

## Sentry에 프로젝트 추가 및 설정
* Sentry에 로그인 후 프로젝트를 생성합니다.
* Client Keys (DSN)메뉴에 있는 DSN키를 복사해둡니다.
  {{< img src="/images/2021/0416/react-dsm.png" alt="react-dsm" link="/images/2021/0416/react-dsm.png">}}
* sentry 패키지 인스톨
  ```
  npm install --save @sentry/react @sentry/tracing
  ```
* index.js에 Sentry를 초기화해주는 코드를 추가합니다.
  ```
  import React from "react";
  import ReactDOM from "react-dom";
  import * as Sentry from "@sentry/react";
  import { Integrations } from "@sentry/tracing";
  import App from "./App";
  import "./index.css";
  Sentry.init({
      // 모든환경에 설정할 경우
      dsn: "https://xxxxxxxxxxxxxxxxxxxxxxxxxxx@xxxxxx.ingest.sentry.io/xxxxx",
      // production환경만 설정할 경우
      dsn: process.env.NODE_ENV === "production"
          ? "https://xxxxxxxxxxxxxxxxxxxxxxxxxxx@xxxxxx.ingest.sentry.io/xxxxx"
          : false,
      integrations: [new Integrations.BrowserTracing()],
      environment: process.env.NODE_ENV,
      tracesSampleRate: 1.0,
  });

  ReactDOM.render(
      <React.StrictMode>
          <App />
      </React.StrictMode>,
      document.getElementById("root")
  );
  ```


* Sentry에 에러를 보내기 위한 설정
  * api실행 중 에러가 발생하면 sentry에 에러를 보내도록 추가했습니다.
    ```
    import * as Sentry from "@sentry/react";
    ...
    ...
    useEffect(() => {
      fetchPost();
    },[]);

    const fetchPost = () => {
        PostDataService.getPost(id)
          .then(response => {
              setCurrentPost(response.data);
            })
          .catch(e => {
              Sentry.captureException(e);
          });
    };
    ```
{{< adsense >}}

## 에러를 발생시켜 Sentry상세페이지 확인
* 404에러를 발생시켜보겠습니다.
{{< img src="/images/2021/0416/react-error.png" alt="react-error" link="/images/2021/0416/react-error.png">}}
* Search by Trace를 클릭하면 에러 추척도 가능하네요.
{{< img src="/images/2021/0416/react-error-trace.png" alt="react-error-trace" link="/images/2021/0416/react-error-trace.png">}}
* Dashboard도 제공하네요.　Custom Dashboards는 Business플랜부터 가능하네요.
{{< img src="/images/2021/0416/react-dashboard.png" alt="react-dashboard" link="/images/2021/0416/react-dashboard.png">}}


사용하고 싶은 기능은 `Team`플랜으로도 충분해서 공식문서를 좀 더 살펴봐야겠습니다.
