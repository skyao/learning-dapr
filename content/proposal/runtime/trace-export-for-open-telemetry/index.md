---
type: blog
date: 2021-02-23
title: "支持OpenTelemetry协议的trace export"
linkTitle: "支持OpenTelemetry协议的trace export"
description: >
  添加使用OpenTelemetry协议的trace export
---

添加使用OpenTelemetry协议的trace export

## Proposal信息

[Support trace export to OpenTelemetry protocol #2836](https://github.com/dapr/dapr/issues/2836)

Now that OpenTelemetry [reached 1.0](https://medium.com/opentelemetry/opentelemetry-specification-v1-0-0-tracing-edition-72dd08936978), we need to support OpenTelemetry protocol. (Currently OpenTelemetry is supported by translating Zipkin protocol with OpenTelemetry Collector).

> 现在OpenTelemetry达到了1.0，我们需要支持OpenTelemetry协议。目前OpenTelemetry是通过翻译Zipkin协议和OpenTelemetry Collector来支持的）。

On the other hand, the OpenTelemetry Go SDK are still going through their [release candidates](https://github.com/orgs/open-telemetry/projects/5). We still need to wait for a 1.0 release to migrate our code to use this SDK, but we can start experimenting with the release candidate now.

> 另一方面，OpenTelemetry Go SDK仍在进行发布候选版本。我们仍然需要等待1.0版本的发布，以迁移我们的代码来使用这个SDK，但我们现在可以开始用候选版本进行尝试。

### 提案讨论

Not yet.
