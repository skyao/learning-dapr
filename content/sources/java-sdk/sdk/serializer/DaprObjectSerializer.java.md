---
title: "DaprObjectSerializer"
linkTitle: "DaprObjectSerializer"
weight: 20
date: 2021-02-24
description: >
  DaprObjectSerializer 接口定义了 dapr 的对象序列化器
---

### 接口定义

DaprObjectSerializer 接口很简单，定义如下：

```java
// 对应用程序的对象进行序列化和反序列化
public interface DaprObjectSerializer {

  // 将给定的对象序列化为byte[].
  byte[] serialize(Object o) throws IOException;

  // 将给定的byte[]反序列化为一个对象。
  <T> T deserialize(byte[] data, TypeRef<T> type) throws IOException;

  // 返回请求的内容类型
  String getContentType();
}
```

getContentType() 方法获知内容的类型，serialize() 和 deserialize() 分别实现序列化和反序列化，即实现对象和 byte[] 的相互转换。

