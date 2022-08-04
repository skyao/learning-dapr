---
title: "rule注解"
linkTitle: "rule注解"
weight: 20
date: 2022-07-24
description: >
  rule 注解用来表述匹配规则
---

`@topic` 注解用来表述匹配规则。

```java
@Documented
@Target(ElementType.ANNOTATION_TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Rule {

  // 用于匹配传入的 cloud event 的通用表达式语言（ Common Expression Language / CEL）表达。
  String match();

  // 规则的优先级，用于排序。最低的数字有更高的优先权。
  int priority();
}
```

以下是 @rule 注解使用的典型例子：

```java
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}",
          rule = @Rule(match = "event.type == \"v2\"", priority = 1))
  @PostMapping(path = "/testingtopicV2")
  public Mono<Void> handleMessageV2(@RequestBody(required = false) CloudEvent cloudEvent) {
    ......
  }
```


