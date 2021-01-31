---
title: "可观测性文档中的Metrics"
linkTitle: "Metrics"
weight: 918
date: 2021-01-31
description: >
  Dapr可观测性文档中的Metrics
---


> 内容节选自：https://docs.dapr.io/developing-applications/building-blocks/observability/metrics/

Dapr公开了一个Prometheus指标端点（metrics endpoint），您可以通过刮削该端点来更好地了解Dapr的行为方式，并针对特定条件设置警报。

### 配置

Dapr系统进程默认启用metrics端点，您可以通过命令行参数`--enable-metrics=false`来禁用它。

默认的metrics端口是9090。可以通过向Daprd传递命令行参数`--metrics-port`来重写这个端口。

如果要禁用Dapr Sidecar中的度量，可以使用 `metrics` spec 配置，并设置 `enabled: false`来禁用Dapr运行时的度量。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing
  namespace: default
spec:
  tracing:
    samplingRate: "1"
  metric:
    enabled: false
```

### Metrics

每个Dapr系统进程都会默认发出Go运行时/进程指标，并有自己的指标

- [Dapr metric list](https://github.com/dapr/dapr/blob/master/docs/development/dapr-metrics.md)

