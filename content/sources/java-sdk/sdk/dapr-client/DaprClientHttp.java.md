---
title: "DaprClientHttp"
linkTitle: "DaprClientHttp"
weight: 30
date: 2021-02-24
description: >
  Dapr Client Http 实现
---

## 类定义

```java
public class DaprClientHttp extends AbstractDaprClient {

  private final DaprHttp client;
  private final boolean isObjectSerializerDefault;
  private final boolean isStateSerializerDefault;

  DaprClientHttp(DaprHttp client, DaprObjectSerializer objectSerializer, DaprObjectSerializer stateSerializer) {
    super(objectSerializer, stateSerializer);
    this.client = client;
    this.isObjectSerializerDefault = objectSerializer.getClass() == DefaultObjectSerializer.class;
    this.isStateSerializerDefault = stateSerializer.getClass() == DefaultObjectSerializer.class;
  }

   DaprClientHttp(DaprHttp client) {
    this(client, new DefaultObjectSerializer(), new DefaultObjectSerializer());
  }
}
```

## client特有的方法实现

### waitForSidecar() 方法

waitForSidecar() 方法通过连接指定的 sidecar ip地址和端口来判断并等待 sidecar 是不是可用。

```java
  public Mono<Void> waitForSidecar(int timeoutInMilliseconds) {
    return Mono.fromRunnable(() -> {
      try {
        NetworkUtils.waitForSocket(Properties.SIDECAR_IP.get(), Properties.HTTP_PORT.get(), timeoutInMilliseconds);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    });
  }
```

### close() 方法

close() 方法是实现 java.lang.AutoCloseable 的要求，DaprClient 继承了这个接口：

```java
  @Override
  public void close() {
    // 简单的关闭 http client
    client.close();
  }
```

## dapr api 方法的实现

### publishEvent()方法

publishEvent()方法主要是两个任务：

1. 组装发送请求的各种参数，包括 http 请求的 method，path，parameters，以及事件序列化后 byte[] 格式的数据
2. 调用 DaprClient 发出 HTTP 请求

```java
@Override
  public Mono<Void> publishEvent(PublishEventRequest request) {
    try {
      String pubsubName = request.getPubsubName();
      String topic = request.getTopic();
      Object data = request.getData();
      Map<String, String> metadata = request.getMetadata();

      if (topic == null || topic.trim().isEmpty()) {
        throw new IllegalArgumentException("Topic name cannot be null or empty.");
      }

      byte[] serializedEvent = objectSerializer.serialize(data);
      // Content-type can be overwritten on a per-request basis.
      // It allows CloudEvents to be handled differently, for example.
      String contentType = request.getContentType();
      if (contentType == null || contentType.isEmpty()) {
        contentType = objectSerializer.getContentType();
      }
      Map<String, String> headers = Collections.singletonMap("content-type", contentType);

      String[] pathSegments = new String[]{ DaprHttp.API_VERSION, "publish", pubsubName, topic };

      Map<String, List<String>> queryArgs = metadataToQueryArgs(metadata);
      return Mono.subscriberContext().flatMap(
          context -> this.client.invokeApi(
              DaprHttp.HttpMethods.POST.name(), pathSegments, queryArgs, serializedEvent, headers, context
          )
      ).then();
    } catch (Exception ex) {
      return DaprException.wrapMono(ex);
    }
  }
```

### shutdown() 方法

注意这个 shutdown() 方法是关闭 sidecar，因此也是需要发送请求到 sidecar 的：

```java
  @Override
  public Mono<Void> shutdown() {
    String[] pathSegments = new String[]{ DaprHttp.API_VERSION, "shutdown" };
    return Mono.subscriberContext().flatMap(
            context -> client.invokeApi(DaprHttp.HttpMethods.POST.name(), pathSegments,
                null, null, context))
        .then();
  }
```

http 请求最终是通过 DaprClient 发出去的。

