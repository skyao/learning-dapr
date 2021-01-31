---
title: "资源绑定的概念"
linkTitle: "概念"
weight: 712
date: 2021-01-31
description: >
  Dapr资源绑定的概念
---


> 内容摘选自：https://github.com/dapr/docs/blob/master/concepts/bindings/README.md

使用绑定，可以用从外部系统来的事件触发应用，或者调用外部系统。绑定允许按需，事件驱动的计算方案，而dapr绑定可以在以下方面帮助开发人员：

- 消除了连接到诸如队列，消息总线等消息系统以及从中进行轮询的复杂性。
- 关注业务逻辑，而不关注与系统交互的实现细节
- 保持代码不受SDK或类库的影响
- 处理重试和故障恢复
- 运行时切换绑定
- 在特定于环境的绑定搭建后，支持可移植应用，无需更改代码

绑定是独立于Dapr运行时开发的。

### 支持的绑定和规格

见原文。

> 疑问：为什么有些没有支持 input，有些没有支持 output？

### 输入绑定

什么是 Input Binding：

> An input binding represents an event resource that Dapr uses to read events from and push to your application.
>
> 输入绑定表示事件资源，Dapr用于从中读取事件并将其推送到应用。

当来自外部资源的事件发生时，输入绑定用于触发应用。可选的有效负载和元数据可能与请求一起发送。

为了接收来自输入绑定的事件：

1. 定义描述绑定类型及其元数据（连接信息等）的组件YAML。
2. 在HTTP端点上监听传入事件，或使用gRPC proto 类库获取传入事件

> 在启动时，Dapr将为所有已经定义的输入绑定发送 `OPTIONS` 请求到应用，并期望 `NOT FOUND (404)` 之外的状态代码，如果应用想要订阅绑定。

#### Input Binding的优势

> 备注：节选自 https://github.com/dapr/docs/tree/master/howto/trigger-app-with-input-binding

使用绑定，可以用来自不同资源的传入事件触发代码，这些资源可以是任何东西：队列，消息管道，云服务，文件系统等。

非常适用于于事件驱动的处理（event-driven processing），数据管道（data pipelines）或仅对事件做出反应并进行进一步处理。

Dapr绑定可以：

- 接收事件而不包含特定的SDK或类库
- 替换绑定而无需更改代码
- 关注业务逻辑，而不是事件资源的实现

具体使用方法，见官方文档：[Create an event-driven app using input bindings](https://github.com/dapr/docs/tree/master/howto/trigger-app-with-input-binding)

### 输出绑定

什么是 Output Binding：

> An output binding represents a resource that Dapr will use invoke and send messages to.
>
> 输出绑定表示Dapr将使用调用(invoke)和向其发送消息的资源。

输出绑定允许用户调用外部资源。可选的有效负载和元数据可以随调用请求一起发送。

为了调用输出绑定：

1. 定义描述绑定类型及其元数据（连接信息等）的组件YAML。
2. 使用HTTP端点或gRPC方法调用绑定，可带有可选有效负载

