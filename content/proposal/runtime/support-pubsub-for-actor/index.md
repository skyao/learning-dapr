---
type: blog
date: 2019-10-02
title: "为Actor提供pubsub支持"
linkTitle: "为Actor提供pubsub支持"
description: >
  为Actor提供pubsub支持
---

为Actor提供pubsub支持

## Proposal信息

[Support pubsub for Actors #501](https://github.com/dapr/dapr/issues/501)

Support pubsub between actors, currently the publisher can be a service or an actor. But when it comes to subscribing, the subscribe is supported at service level. We need to extend this to support subscribe at an Actor level. An actor should be able to subscribe to a message of type "Message_One" with its actor id and when that message is published a specific method is invoked on that particular actor id. If multiple actor ids have subscribed for that message then method will be invoked for all the subscribed actor ids.

> 支持 actor 之间的pubsub，目前发布者可以是服务，也可以是Actor。但是当涉及到订阅的时候，订阅的支持是在服务级别上的。我们需要扩展这个功能来支持 Actor 级别的订阅。Actor应该能够用它的 actor id 订阅类型为 "Message_One" 的消息，当这个消息被发布时，这个特定的actor id上的指定方法将被调用。如果有多个 actor id 订阅了该消息，那么所有订阅的actor id都会被调用该方法。





