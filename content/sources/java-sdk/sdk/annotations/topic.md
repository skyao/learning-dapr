---
title: "topic注解"
linkTitle: "topic注解"
weight: 10
date: 2021-02-24
description: >
  topic 注解提供对 subscribe 的支持
---

`@topic` 注解用来订阅某个主题， pubsubName, name, metadata 分别对应 dapr pub/sub API 中的 pubsubName， topic，metadata 字段：


```java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Topic {
  String name();
  String pubsubName();
  String metadata() default "{}";

  // 用于匹配传入的 cloud event 的规则。
  Rule rule() default @Rule(match = "", priority = 0);
}
```

以下是 @topic 注解使用的典型例子：

```java
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}")
  @PostMapping(path = "/testingtopic")
  public Mono<Void> handleMessage(@RequestBody(required = false) CloudEvent<?> cloudEvent) {
    ......
  }
```
