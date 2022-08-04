---
title: "DaprClientGrpc"
linkTitle: "DaprClientGrpc"
weight: 40
date: 2021-02-24
description: >
  Dapr Client gRPC 实现
---

## 类定义

```java

public class DaprClientGrpc extends AbstractDaprClient {

  private Closeable channel;

  private DaprGrpc.DaprStub asyncStub;

  DaprClientGrpc(
      Closeable closeableChannel,
      DaprGrpc.DaprStub asyncStub,
      DaprObjectSerializer objectSerializer,
      DaprObjectSerializer stateSerializer) {
    super(objectSerializer, stateSerializer);
    this.channel = closeableChannel;
    this.asyncStub = intercept(asyncStub);
  }

}
```

## client特有的方法实现

### waitForSidecar() 方法

waitForSidecar() 方法通过连接指定的 sidecar ip地址和端口来判断并等待 sidecar 是不是可用。

和 HTTP 的实现差别只是端口不同。

```java
  @Override
  public Mono<Void> waitForSidecar(int timeoutInMilliseconds) {
    return Mono.fromRunnable(() -> {
      try {
        NetworkUtils.waitForSocket(Properties.SIDECAR_IP.get(), Properties.GRPC_PORT.get(), timeoutInMilliseconds);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    });
  }
```

### close() 方法

close() 方法是实现 java.lang.AutoCloseable 的要求，DaprClient 继承了这个接口：

```java
  public void close() throws Exception {
    if (channel != null) {
      DaprException.wrap(() -> {
        // 关闭channel
        channel.close();
        return true;
      }).call();
    }
  }
```


## dapr api 方法的实现

### publishEvent()方法 

publishEvent()方法主要是两个任务：

1. 组装发送 grpc 请求的各种参数，构建 PublishEventRequest 请求对象
2. 调用 gRPC asyncStub 的对应方法发出 gRPC 请求

```java
@Override
  public Mono<Void> publishEvent(PublishEventRequest request) {
    try {
      String pubsubName = request.getPubsubName();
      String topic = request.getTopic();
      Object data = request.getData();
      DaprProtos.PublishEventRequest.Builder envelopeBuilder = DaprProtos.PublishEventRequest.newBuilder()
          .setTopic(topic)
          .setPubsubName(pubsubName)
          .setData(ByteString.copyFrom(objectSerializer.serialize(data)));

      // Content-type can be overwritten on a per-request basis.
      // It allows CloudEvents to be handled differently, for example.
      String contentType = request.getContentType();
      if (contentType == null || contentType.isEmpty()) {
        contentType = objectSerializer.getContentType();
      }
      envelopeBuilder.setDataContentType(contentType);

      Map<String, String> metadata = request.getMetadata();
      if (metadata != null) {
        envelopeBuilder.putAllMetadata(metadata);
      }

      return Mono.subscriberContext().flatMap(
          context ->
              this.<Empty>createMono(
                  it -> intercept(context, asyncStub).publishEvent(envelopeBuilder.build(), it)
              )
      ).then();
    } catch (Exception ex) {
      return DaprException.wrapMono(ex);
    }
  }
```