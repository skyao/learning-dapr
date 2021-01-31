---
title: "发布订阅的概念"
linkTitle: "概念"
weight: 602
date: 2021-01-31
description: >
  Dapr的发布订阅（pub-sub）的概念
---

> 内容摘选自：https://github.com/dapr/docs/blob/master/concepts/publish-subscribe-messaging/README.md

Dapr使开发人员可以用发布/订阅模式设计其应用，使用消息代理，将发布者/生产者彼此解耦，并通过发送和接收与名称空间关联的消息（通常以 topic 的形式）进行通信。

这使事件生产者可以将消息发送给未运行的消费者，并且消费者可以根据对 topic 的订阅来接收消息。

Dapr 提供“至少一次”消息传递保证，并与各种消息代理实现集成。这些实现是可插拔的，并在Dapr运行时之外的 components-contrib 中开发。

### 行为与保证

Dapr保证消息传递的至少一次（At-Least-Once）语义。也就是说，当应用程序使用 Publish/Subscribe API将消息发布到 topic 时，当来自该端点的响应状态代码`200`，或者使用gRPC而不返回错误时，它可以假定消息至少传递给任何订阅者一次，。

Dapr 承担了处理诸如消费者分组和消费者分组内部的多个实例之类概念的重担。

### App ID

Dapr有一个 `app id` 的概念。在Kubernetes中是使用`dapr.io/id`注释指定的，而 app-id 在Dapr CLI中使用 flag 指定的。Dapr要求为每个应用分配一个ID。

当同一应用ID的多个实例订阅同一个 topic 时，Dapr将确保仅将消息传递给一个实例。如果两个具有不同ID的不同应用订阅了一个 topic ，则每个应用中的至少一个实例将收到同一消息的副本。

### Cloud Events

Dapr遵循 [Cloud Events 0.3 规范](https://github.com/cloudevents/spec/tree/v0.3) ，并将发送到 topic 的所有有效负载包装在 Cloud Events 封套内。

Dapr实现了 Cloud Events规范中的以下字段：

- `id`
- `source`
- `specversion`
- `type`
- `datacontenttype` （可选的）