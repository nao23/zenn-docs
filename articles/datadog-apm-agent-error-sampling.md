---
title: 【Datadog APM】Datadog Agent によるエラーサンプリング検証
type: tech
topics: ["datadog", "opentelemetry"]
emoji: 🐶
published: true
---

Datadog APMを使って分散トレースを可視化・分析できるようにすることを検証しており、その中でDatadog Agent内のエラーサンプリング機能の検証で少々手こずったところがあったので、メモとして記録しておきます。

# 検証環境
- Pythonで書いたWeb API サーバアプリケーションをOpenTelemetryで計装
- AWS ECS Fargateでホスティング
- Datadog Agentをサイドカーで起動し、OpenTelemeryからトレースを受け取ってAPMに転送
  - Datadog Agent のバージョン: 7.61.0

# 設定

## 基本設定

まず、基本設定として以下の環境変数を設定し、OpenTelemetry SDKからトレースを受け取ってDatadog APMに転送するようにします。

```
ECS_FARGATE=true
DD_APM_ENABLED=true
DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_GRPC_ENDPOINT=localhost:4317
```

## エラーサンプリングのみが行われるようにする設定

次に今回はエラーサンプリングの検証をしたいので、エラーサンプリングだけが行われるように設定します。

ドキュメントを読むと、まずヘッドベースサンプリングが行われ、そこで漏れたエラートレースがエラーサンプリングで補足されると書いてあるので、ヘッドベースサンプリングをOFFにする必要がありますが、その方法がドキュメントからはよくわかりませんでした。

最初 `DD_APM_MAX_TPS` を0に設定すれば良いかと思い試しましたが機能せず、ドキュメントを改めて読むとこの設定値はDatadogのトレースライブラリを利用した場合のみ機能するもので、OpenTelemery SDKを利用した場合は機能しない設定値でした。
https://docs.datadoghq.com/ja/tracing/trace_pipeline/ingestion_mechanisms/


サポートに問い合わせて色々見ていただいたところ、最終的に以下の環境変数を追加で設定することでエラーサンプリングのみを行うことができました。

```
DD_APM_PROBABILISTIC_SAMPLER_ENABLED=true
DD_APM_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE=0
```

### なぜこのような設定にする必要があるか
追加の設定の意味としては probabilistic sampler を有効（デフォルト: 無効）にし、その割合を0%にするということですが、なぜこのような設定をする必要があるかというと

- Datadog Agent 内ではデフォルトで `DD_OTLP_CONFIG_TRACES_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE` の設定値に基づく probabilistic samplerが動作している

  - `DD_OTLP_CONFIG_TRACES_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE` (デフォルト値: 100)は0が設定できない

- `DD_APM_PROBABILISTIC_SAMPLER_ENABLED=true` を設定すると、`DD_OTLP_CONFIG_TRACES_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE`の動作を上書きし、`DD_APM_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE`によって取り込み率が決まるprobabilistic samplerが動作する
  - `DD_APM_PROBABILISTIC_SAMPLER_SAMPLING_PERCENTAGE` は0が設定できる

ということのようです。

# 挙動
## 厳密なサンプリングレートは設定できない

エラーサンプリングでは `DD_APM_ERROR_TPS` で取り込み率を設定できますが、この設定値はあくまで目標値なので厳密にはこの値にはならない点に注意が必要です。

わざとエラートレースを出すAPIエンドポイントを用意し、そこに対して秒間10リクエスト程度の負荷をかけてどのくらいのトレースが取り込まれるかを見たところ、最大5トレース/sec 程度取り込まれることを確認しました。