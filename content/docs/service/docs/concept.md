---
title: "服务调用的概念"
linkTitle: "概念"
weight: 512
date: 2021-01-29
description: >
  Dapr服务调用的概念
---

> 内容摘选自：https://github.com/dapr/docs/tree/master/concepts/service-invocation

使用服务调用API，微服务可以使用标准协议（目前支持gRPC或HTTP）找到并可靠地与系统中的其他微服务进行通信。

以下是Dapr的服务调用系统工作原理的高层次概述:

![Service Invocation Diagram](images/service-invocation.png)

1. 服务A发起了针对服务B的http/gRPC调用。该调用转到本地的Dapr Sidecar。
2. Dapr发现服务B的位置，并将消息转发到服务B的Dapr sidecar
3. 服务B的Dapr sidecar将请求转发给服务B。服务B执行其相应的业务逻辑。
4. 服务B给服务A发送响应。该响应转到服务B的 sidecar。
5. Dapr将响应转发给Service A的Dapr边车。
6. 服务A收到响应。

作为上述所有内容的一个示例，假设我们具有以下示例中描述的应用集合，其中python应用程序调用Node.js应用：
https://github.com/dapr/samples/blob/master/2.hello-kubernetes/README.md

在这种情况下，python应用程序将是上面的“服务A”，而Node.js应用程序将是“服务B”。

下面描述了在此示例的上下文中的步骤1-6：

1. 假设Node.js应用的Dapr应用id为 "nodeapp"，如示例中。python应用通过发 Post 请求到 http://localhost:3500/v1.0/invoke/nodeapp/method/neworder，调用Node.js应用的neworder方法，首先进入python应用的本地Dapr sidecar。
2. Dapr发现Node.js应用的位置，并将其转发给Node.js应用的sidecar。
3. Node.js应用的sidecar将请求转发到Node.js应用。Node.js应用执行其业务逻辑，正如示例中所描述的那样，它是将传入的消息记录下来，然后将订单ID持久化到Redis中（上图中没有显示）。


步骤4-5与上面的列表相同。


