---
title: "如何调用远程服务"
linkTitle: "如何调用远程服务"
weight: 512
date: 2021-01-29
description: >
  Dapr如何调用远程服务
---

> https://github.com/dapr/docs/tree/master/howto/invoke-and-discover-services

在很多有多个服务需要相互通信的环境中，开发人员经常会问自己以下问题：

- 如何发现和调用不同的服务？
- 如何处理重试和瞬时错误？
- 如何正确使用分布式跟踪来查看调用图？

Dapr通过提供一个端点，使开发人员能够克服这些挑战，该端点的作用是将反向代理与内置的服务发现相结合，同时利用内置的分布式跟踪和错误处理。

## 1. 为您的服务选择一个ID

Dapr允许您为您的应用程序分配一个全局的、唯一的ID。

这个ID封装了您的应用程序的状态，无论它可能有多少个实例。

### 使用Dapr CLI设置一个ID

在单机模式下，设置 --app-id 标志：

```
dapr run --app-id cart --app-port 5000 python app.py
```

### 使用Kubernetes设置一个ID

在Kubernetes中，在你的pod上设置 dapr.io/id 注解。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  namespace: default
  labels:
    app: python-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
      annotations:
        dapr.io/enabled: "true"
        dapr.io/id: "cart"
        dapr.io/port: "5000"
...
```

## 在代码中调用服务

Dapr使用的是sidecar，非中央化架构。要使用Dapr调用一个应用程序，你可以在你的集群/环境中的任何Dapr实例上使用 invoke 端点。

sidecar编程模型鼓励每个应用程序与自己的Dapr实例对话。Dapr实例之间相互发现和通信。

> 注意：下面是一个购物车应用的Python例子。它可以用任何编程语言编写

```python
from flask import Flask
app = Flask(__name__)

@app.route('/add', methods=['POST'])
def add():
    return "Added!"

if __name__ == '__main__':
    app.run()
```

这个Python应用通过 /add 端点暴露了一个add()方法。

### 用curl调用

```
curl http://localhost:3500/v1.0/invoke/cart/method/add -X POST
```

由于add端点是一个 "POST" 方法，我们在curl命令中使用了 -X POST。

要调用一个'GET'端点：

```
curl http://localhost:3500/v1.0/invoke/cart/method/add
```

要调用'DELETE'端点：

```
curl http://localhost:3500/v1.0/invoke/cart/method/add -X DELETE
```

Dapr将其调用的服务返回的任何有效载荷放在HTTP响应的body中。

## 概述

上面的例子向您展示了如何直接调用在我们的环境中运行的不同服务，本地或Kubernetes中。Dapr输出指标和跟踪信息，允许您可视化服务之间的调用图，记录错误并可选地记录有效载荷体。



