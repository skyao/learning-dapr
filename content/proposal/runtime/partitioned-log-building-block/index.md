---
type: blog
date: 2021-02-19
title: "添加分区日志构建块"
linkTitle: "添加分区日志构建块"
description: >
  在对事件驱动系统进行建模时，分区日志抽象是非常强大的
---

在对事件驱动系统进行建模时，分区日志抽象是非常强大的

## Proposal信息

[Add 'partitioned log' building block](https://github.com/dapr/dapr/issues/100)

The partitioned log abstraction is incredibly powerful when modeling an event driven system. It has similar semantics as pub/sub in the sense that services publish events and interested parties consume them. However, there are key differences:

> 在对事件驱动系统进行建模时，分区日志抽象是非常强大的，它与pub/sub的语义类似，即服务发布事件，相关方消费它们。然而，有一些关键的区别。

- the events are meant to be the source of truth for business decisions in the domain and therefor need to live forever in the log
- the consumers of the logs must be able to consume at their own time and from their own desired offsets
- the publisher needs to be able to specify how its events are partitioned and
- the partitions must guarantee ordering of events.

> - 事件是为了成为该领域业务决策的真相来源，因此需要永远存在于日志中。
> - 日志的消费者必须能够在自己的时间和从自己所需的 offset 中进行消费
> - 发布者需要能够指定其事件如何被分区和处理。
> - 分区必须保证事件的排序。

Practically, I'm aware of two implementations of partitioned logs: Kafka and Pulsar. Now while dapr already has two ways of using Kafka via pub/sub and I/O bindings, these aren't really the correct abstractions. As an example, Kafka could be replaced by rabbitmq when using pub/sub, which has none of the properties of partitioned logs. And when using the Kafka binding, you're not really programming against the partitioned log abstraction, but rather just concretely against Kafka.

> 实际上，我知道有两种分区日志的实现。Kafka和Pulsar。现在虽然dapr已经有两种的方式：通过pub/sub和I/O绑定，但这些并不是真正正确的抽象。举个例子，当使用pub/sub时，Kafka可以用rabbitmq代替，它没有任何分区日志的属性。而当使用Kafka绑定时，你并没有真正针对分区日志抽象进行编程，而只是具体针对Kafka进行编程。

I therefore propose adding a new 'partitioned log' building block to dapr which is similar to the pub/sub building block except the components implementing that building block must have the partitioned log properties and the API must also include a way to specify the partitioning of events for publishers and a way to get the partition offset of events by consumers. This is of interest when using the partitioned log abstraction because the partition guarantees ordering and thus the offset can be viewed as a version attribute which can be very useful.

> 因此，我建议在dapr中添加一个新的 "分区日志" 构建块，它与pub/sub构建块类似，只是实现该构建块的组件必须具有分区日志属性，而且API还必须包括一种为发布者指定事件分区的方法和一种为消费者获取事件分区offset的方法。在使用分区日志抽象时，这一点很有意义，因为分区保证了排序，因此offset可以被看作是一个版本属性，这一点非常有用。

### 提案讨论
