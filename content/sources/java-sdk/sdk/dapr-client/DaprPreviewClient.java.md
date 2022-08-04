---
title: "DaprPreviewClient"
linkTitle: "DaprPreviewClient"
weight: 15
date: 2021-02-24
description: >
  DaprPreviewClient 接口用于定义 preview 和 alpha 的 API
---

DaprPreviewClient 接口定义，目前只有新增的 configuration api 的方法和 state query 的方法：


```java
// 无论需要何种GRPC或HTTP客户端实现，都可以使用通用客户端适配器。
public interface DaprPreviewClient extends AutoCloseable {

  Mono<ConfigurationItem> getConfiguration(String storeName, String key);

  Flux<List<ConfigurationItem>> subscribeToConfiguration(String storeName, String... keys);

  <T> Mono<QueryStateResponse<T>> queryState(String storeName, String query, TypeRef<T> type);
}
```

> 备注：distribuyted lock 的方法还没有加上来，估计是还没有开始实现。
