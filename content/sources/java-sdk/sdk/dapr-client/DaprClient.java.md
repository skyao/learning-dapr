---
title: "DaprClient"
linkTitle: "DaprClient"
weight: 10
date: 2021-02-24
description: >
  DaprClient 接口定义
---


```java
// 无论需要何种GRPC或HTTP客户端实现，都可以使用通用客户端适配器。
public interface DaprClient extends AutoCloseable {

  Mono<Void> waitForSidecar(int timeoutInMilliseconds);

  Mono<Void> shutdown();
}
```

其他方法都是和 dapr api 相关的方法，然后所有的方法都是实现了 reactive 风格，如：

```java
  Mono<Void> publishEvent(String pubsubName, String topicName, Object data);

  Mono<Void> publishEvent(String pubsubName, String topicName, Object data, Map<String, String> metadata);

  Mono<Void> publishEvent(PublishEventRequest request);
```