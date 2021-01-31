---
title: "服务调用的参考文档"
linkTitle: "服务调用的参考文档"
weight: 512
date: 2021-01-29
description: >
  Dapr 服务调用的参考文档
---

> https://github.com/dapr/docs/blob/master/reference/api/service_invocation_api.md

TBD：gRPC的参考文档呢？调用的例子倒是有一个：https://github.com/dapr/docs/blob/master/howto/create-grpc-app

----------

Dapr为用户提供了调用其他具有唯一id的应用程序的能力。该功能允许应用程序通过命名标识符相互交互，并将服务发现的负担放在Dapr运行时。

## 调用远程dapr应用的方法

这个端点可以在另一个启用Dapr的应用程序中调用方法：

### HTTP Request

```HTTP
POST/GET/PUT/DELETE http://localhost:<daprPort>/v1.0/invoke/<appId>/method/<method-name>
```

### HTTP Response Codes

当服务用Dapr调用另一个服务时，被调用服务的状态码将返回给调用者。如果出现网络错误或其他短暂的错误，Dapr将返回一个500错误，并附上详细的错误信息。

如果用户通过HTTP调用Dapr与启用了gRPC的服务对话，被调用的gRPC服务将返回500错误，成功响应将返回200OK。

| Code | Description    |
| ---- | -------------- |
| 500  | Request failed |

### URL Parameters

| Parameter   | Description                               |
| ----------- | ----------------------------------------- |
| daprPort    | the Dapr port                             |
| appId       | the App ID associated with the remote app |
| method-name | 调用的远程应用的方法或url的名称           |

### Request Contents

In the request you can pass along headers:

```
{
  "Content-Type": "application/json"
}
```

Within the body of the request place the data you want to send to the service:

```
{
  "arg1": 10,
  "arg2": 23,
  "operator": "+"
}
```

### 被调用服务收到的请求

一旦您的服务代码在另一个启用了 Dapr 的应用程序中调用了一个方法，Dapr 将把请求连同头和主体一起发送到 <method-name> 端点上的应用程序。

被调用的Dapr应用需要监听并响应该端点上的请求。

### Example

你可以通过发送以下内容来调用mathService服务的add方法。

```http
curl http://localhost:3500/v1.0/invoke/mathService/method/add \
  -H "Content-Type: application/json"
  -d '{ "arg1": 10, "arg2": 23
```

mathService服务需要在 /add 端点上监听，以接收和处理该请求: 

对于Node应用来说，这将是这样的：

```go
app.post('/add', (req, res) => {
  let args = req.body;
  const [operandOne, operandTwo] = [Number(args['arg1']), Number(args['arg2'])];
  
  let result = operandOne / operandTwo;
  res.send(result.toString());
});

app.listen(port, () => console.log(`Listening on port ${port}!`));
```

> 远程端点的响应将在请求体中返回。

如果服务监听一个更多的嵌套路径(例如 `/api/v1/add`)，Dapr实现了完整的反向代理，所以可以将所有必要的路径片段附加到请求URL中，就像这样:

http://localhost:3500/v1.0/invoke/mathService/method/api/v1/add