---
title: "可观测性文档的W3C追踪上下文概述"
linkTitle: "W3C追踪上下文概述"
weight: 915
date: 2021-01-31
description: >
  Dapr可观测性文档中的W3C追踪上下文概述
---


> 内容节选自：https://docs.dapr.io/developing-applications/building-blocks/observability/w3c-tracing/w3c-tracing-overview/

**使用Dapr进行W3C追踪的背景和场景**

## 介绍

Dapr使用W3C跟踪上下文对服务调用和pub/sub消息进行分布式跟踪。Dapr主要完成了生成和传播跟踪上下文信息的所有繁重工作，这些信息可以被发送到许多不同的诊断工具进行可视化和查询。只有极少数情况下，作为开发者，你需要传播或生成tracing header。

## 背景

分布式跟踪是一种由跟踪工具实现的方法，用于跟踪、分析和调试跨多个软件组件的事务。通常情况下，分布式跟踪会遍历一个以上的服务，这就要求它具有唯一的标识性。跟踪上下文传播将这种唯一标识传递出去。

在过去，跟踪上下文传播通常由每个不同的跟踪供应商单独实现。在多厂商的环境中，这会造成互操作性的问题，比如：

- 由于没有共享的唯一标识符，不同追踪供应商收集的tracing无法相互关联。
- 跨越不同追踪供应商边界的trace无法传播，因为没有统一协商的标识符集可以被转发。
- 厂商特定的元数据可能会被中介机构放弃
- 云平台厂商、中间商和服务商，由于没有标准可循，不能保证支持追踪上下文传播。

在过去，这些问题并没有产生重大影响，因为大多数应用程序由单一的跟踪供应商监控，并停留在单一平台供应商的边界内。今天，越来越多的应用是分布式的，并利用了多个中间件服务和云平台。

现代应用的这种转变呼唤一个分布式的跟踪上下文传播标准。[W3C跟踪上下文规范](https://www.w3.org/TR/trace-context/)为跟踪上下文传播数据的交换定义了一种普遍认同的格式--称为跟踪上下文。trace context通过以下方式解决了上述问题：

- 为单个跟踪和请求提供独特的标识符，允许将多个供应商的跟踪数据链接在一起。
- 提供一个约定俗成的机制，以转发特定供应商的跟踪数据，并避免在多个跟踪工具参与单一交易时出现跟踪中断的情况。
- 提供一个中间商、平台和硬件提供商可以支持的行业标准。

传播跟踪数据的统一方法提高了分布式应用行为的可视性，便于问题和性能分析。

## 场景

有两种情况下，你需要了解跟踪是如何被使用的：

1. Dapr生成并在服务之间传播跟踪上下文。
2. Dapr生成跟踪上下文，你需要将跟踪上下文传播给另一个服务，或者你生成跟踪上下文，Dapr将跟踪上下文传播给一个服务。

### Dapr在服务之间生成和传播跟踪上下文

在这些情况下，Dapr为你做了所有的工作。你不需要创建和传播任何跟踪头。Dapr会负责创建所有的跟踪头并传播它们。让我们通过实例来了解一下这些场景。

1. 单个服务调用（`service A -> service B`）

    Dapr在服务A中生成跟踪头，这些跟踪头从服务A传播到服务B。

2. 多个顺序的服务调用 ( `service A -> service B -> service C`)

    Dapr在服务A请求开始时生成跟踪头，这些跟踪头从服务A->服务B->服务C，以此类推传播到更多的启用了Dapr的服务。

3. 请求来自外部端点（例如从网关服务到启用Dapr的服务A的请求）

    Dapr 在服务 A 中生成跟踪头，这些跟踪头从服务 A 进一步传播到启用 Dapr 的服务服务 A->服务 B ->服务 C。和上面的场景2类似。

4. Pub/sub消息 Dapr在发布的消息主题中生成跟踪头，这些跟踪头被传播到该主题上的任何监听服务。

### 你需要在服务之间传播或生成跟踪上下文

在这些情况下，Dapr为你做了一些工作，你需要创建或传播跟踪头。

1. 从单个服务对不同服务的多次服务调用

    当你从一个服务中调用多个服务时，比如像这样从服务A中调用，你需要传播跟踪头。

    ```
     service A -> service B
     [ .. some code logic ..]
     service A -> service C
     [ .. some code logic ..]
     service A -> service D
     [ .. some code logic ..]
    ```
    
    在这种情况下，当服务A第一次调用服务B时，Dapr会在服务A中生成跟踪头，然后将这些跟踪头传播给服务B，这些跟踪头在服务B的响应中作为响应头的一部分返回。然而你需要将返回的跟踪上下文传播给下一个服务C和服务D，因为Dapr不知道你要重用同一个头。
    
    要了解如何从响应中提取跟踪头，并将跟踪头添加到请求中，请参阅 [如何使用跟踪上下文]( https://docs.dapr.io/developing-applications/building-blocks/observability/w3c-tracing/) 一文。
    
2. 你选择了生成自己的跟踪上下文头。这是很少会遇到的。在某些情况下，您可能会特别选择将 W3C 跟踪头添加到服务调用中，例如，如果您有一个现有的应用程序目前没有使用 Dapr。在这种情况下，Dapr仍然会为您传播跟踪上下文头。如果你决定自己生成跟踪头，有三种方法可以实现：

    1. 您可以使用行业标准的 OpenCensus/OpenTelemetry SDK 来生成跟踪头，并将这些跟踪头传递给启用 Dapr 的服务。这是首选的建议。

    2. 你可以使用供应商SDK来生成W3C跟踪头，如DynaTrace SDK，并将这些跟踪头传递给启用Dapr的服务。

    3. 你可以按照W3C跟踪上下文规范手工制作一个跟踪上下文，并将这些跟踪头传递给启用Dapr的服务。

## W3C trace headers

这些是由Dapr为HTTP和gRPC生成和传播的特定的跟踪上下文头。

### 跟踪上下文的HTTP header格式

当将HTTP响应的跟踪上下文头传播到HTTP请求时，你需要复制这些头。

#### Traceparent Header

traceparent header在跟踪系统中以通用的格式表示传入的请求，所有厂商都能理解。下面是一个traceparent header的例子：

`traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`

traceparent字段在 [这里](https://www.w3.org/TR/trace-context/#traceparent-header) 有详细说明

#### Tracestate Header

tracestate头 header包含了可能是特定于厂商的格式的parent。

`tracestate: congo=t61rcWkgMzE`

[这里](https://www.w3.org/TR/trace-context/#tracestate-header) 是详细的 tracestate 字段的说明。

### 跟踪上下文的gRPC header格式

在gRPC API调用中，跟踪上下文是通过 `grpc-trace-bin` heaer传递的。

