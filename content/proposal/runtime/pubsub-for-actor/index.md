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

### 提案内容

Support pubsub between actors, currently the publisher can be a service or an actor. But when it comes to subscribing, the subscribe is supported at service level. We need to extend this to support subscribe at an Actor level. An actor should be able to subscribe to a message of type "Message_One" with its actor id and when that message is published a specific method is invoked on that particular actor id. If multiple actor ids have subscribed for that message then method will be invoked for all the subscribed actor ids.

> 支持 actor 之间的pubsub，目前发布者可以是服务，也可以是Actor。但是当涉及到订阅的时候，订阅的支持是在服务级别上的。我们需要扩展这个功能来支持 Actor 级别的订阅。Actor应该能够用它的 actor id 订阅类型为 "Message_One" 的消息，当这个消息被发布时，这个特定的actor id上的指定方法将被调用。如果有多个 actor id 订阅了该消息，那么所有订阅的actor id都会被调用该方法。

### 提案的讨论

Given the unique need for some kind of mapping between inbound event and actor (based on the event type and and some kind of logic to derive ID to create/activate actor), I wonder if this possibly better handled in new kind of binding that can be configured with ref to pubsub binding name as a "source" in metadata. Trying to handle this in PubSub construct could potentially add complexity to the simple pubsub construct we support now.

> 考虑到在入站事件和 actor 之间的某种映射的独特需求（基于事件类型和某种逻辑来推导ID以创建/激活 actor），我想知道这是否可能在新的绑定中得到更好的处理，这种绑定可以在元数据中配置引用pubsub绑定名作为 "source"。试图在 PubSub 构造中处理这个问题，可能会给我们现在支持的简单 pubsub 构造增加复杂性。

With the proposal for Actors pubsub, it will not require logic to determine id from the event. The Actors layer in dapr runtime will save that information and will build on top of Dapr's pubsub building block.

> 有了Actors pubsub的提案，它将不需要从事件中确定id的逻辑。dapr运行时的Actors层将保存这些信息，并将建立在Dapr的pubsub构件之上。

Yeah, but this proposal assumes the actor can first subscribe to events only after it gets activated via service invocation.

[@mchmarny](https://github.com/mchmarny) is talking about the scenario where event come in (either through bindings or Pub/Sub) that need to trigger an actor irrespective of it being created earlier.

> 是的，但这个提案假设 actor 只有在通过服务调用被激活后才能首先订阅事件。
>
> @mchmarny说的是事件进来的情况（无论是通过绑定还是Pub/Sub），这些事件需要触发一个actor，而不管它是否是早期创建的。

Yes, these are 2 scenarios.

> 是的，这是两个场景

### 其他

从这个proposal中分离出另外一个proposal

[Support input bindings for actors #1601](https://github.com/dapr/dapr/issues/1601)