---
title: "发布订阅的参考文档"
linkTitle: "参考文档"
weight: 603
date: 2021-01-31
description: >
  Dapr的发布订阅（pub-sub）的参考文档
---


详细见：

https://github.com/dapr/docs/blob/master/reference/api/pubsub.md

发布消息到topic，只要发一个HTTP POST 请求到 dapr sidecar：

```
POST http://localhost:<daprPort>/v1.0/publish/<topic>
```

要接受消息，首先应用在这个地址接受 sidecar 请求，告知 dapr 自己要订阅的 topic：

```
GET http://localhost:<appPort>/dapr/subscribe

应答如：

"["TopicA","TopicB"]"
```

然后应用在这个地址接受sidecar 转发的订阅消息：

```
POST http://localhost:<appPort>/TopicA
```

### 消息封套

Dapr Pub/Sub 遵守 Cloud Events  0.3 版本。