---
title: "DefaultObjectSerializer"
linkTitle: "DefaultObjectSerializer"
weight: 20
date: 2021-02-24
description: >
  DefaultObjectSerializer 是 dapr 的默认对象序列化器
---

DefaultObjectSerializer 继承自 ObjectSerializer, serialize 和 deserialize 都只是代理给 ObjectSerializer ，而 getContentType() 方法则 hard code 为返回 "application/json"：

```java
public class DefaultObjectSerializer extends ObjectSerializer implements DaprObjectSerializer {

  @Override
  public byte[] serialize(Object o) throws IOException {
    return super.serialize(o);
  }

  @Override
  public <T> T deserialize(byte[] data, TypeRef<T> type) throws IOException {
    return super.deserialize(data, type);
  }

  @Override
  public String getContentType() {
    return "application/json";
  }
}
```

