---
title: "Sentryをreact.jsプロジェクトへ適用する"
date: "2021-04-16"
tags: ["Sentry", "React.js"]
author: "bokyung"
---

 [公式サイト](https://docs.sentry.io/platforms/javascript/guides/react/)の見ながらreact.jsプロジェクトにSentryを適用してみました。

## バージョン
* react 17.0.1
* Sentry
  * @sentry/react 6.2.5
  * @sentry/tracing 6.2.5 

## Sentryへプロジェクト追加及び設定
* Sentryへログイン後、プロジェクト作成。
* Client Keys (DSN)メニューに書いてある、DSNキーをコピーしておきます。
  {{< img src="/images/2021/0416/react-dsm.png" alt="react-dsm" link="/images/2021/0416/react-dsm.png">}}
* sentry パッケージインストール
  ```
  npm install --save @sentry/react @sentry/tracing
  ```
* index.jsにSentryを初期化するコードを追加します。
  ```
    import React from "react";
    import ReactDOM from "react-dom";
    import * as Sentry from "@sentry/react";
    import { Integrations } from "@sentry/tracing";
    import App from "./App";
    import "./index.css";
    Sentry.init({
        // 全ての環境に設定時
        dsn: "https://xxxxxxxxxxxxxxxxxxxxxxxxxxx@xxxxxx.ingest.sentry.io/xxxxx",
        // productionのみ設定時
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
* Sentryへエラーを送るための設定
  * api実行時、エラーが発生したらsentryにエラーを送るように追加しました。
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

## エラーを発生させ、Sentry側を確認
* 404エラーを発生させてみました。
{{< img src="/images/2021/0416/react-error.png" alt="react-error" link="/images/2021/0416/react-error.png">}}
* Search by Traceをクリックするとエラートレースも可能です。
{{< img src="/images/2021/0416/react-error-trace.png" alt="react-error-trace" link="/images/2021/0416/react-error-trace.png">}}
* Dashboardも提供しています。カスタムDashboardはBusinessプランから使えるそうです。
{{< img src="/images/2021/0416/react-dashboard.png" alt="react-dashboard" link="/images/2021/0416/react-dashboard.png">}}


利用したい機能は `Team`プランでも十分に使えそうなので、公式ドキュメントの細かい部分を見てみたいと思います。
