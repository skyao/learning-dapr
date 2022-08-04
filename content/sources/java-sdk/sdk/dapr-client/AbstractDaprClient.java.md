---
title: "AbstractDaprClient"
linkTitle: "AbstractDaprClient"
weight: 20
date: 2021-02-24
description: >
  AbstractDaprClient 抽象基类实现
---


```java
// 抽象类，具有客户端实现之间共同的便利方法。
abstract class AbstractDaprClient implements DaprClient, DaprPreviewClient {
  // 这里还是写死了 jackson！
  // TBD： 看下是哪里在用
  protected static final ObjectMapper JSON_REQUEST_MAPPER = new ObjectMapper();

  protected DaprObjectSerializer objectSerializer;

  protected DaprObjectSerializer stateSerializer;

    AbstractDaprClient(
      DaprObjectSerializer objectSerializer,
      DaprObjectSerializer stateSerializer) {
    this.objectSerializer = objectSerializer;
    this.stateSerializer = stateSerializer;
  }
}
```

其他都方法实现基本都是一些代理方法，没有实质性内容，实际实现都应该在子类中实现。

```java
  @Override
  public Mono<Void> publishEvent(String pubsubName, String topicName, Object data) {
    return this.publishEvent(pubsubName, topicName, data, null);
  }

    @Override
  public Mono<Void> publishEvent(String pubsubName, String topicName, Object data, Map<String, String> metadata) {
    PublishEventRequest req = new PublishEventRequest(pubsubName, topicName, data)
        .setMetadata(metadata);
    return this.publishEvent(req).then();
  }
```

这些方法重载可以理解成一些语法糖，可以不用构造复杂的请求对象如 PublishEventRequest 就可以方便的直接使用而已。