---
title: "流程概述"
linkTitle: "概述"
weight: 1
date: 2022-04-21
description: >
  Dapr发布的流程和API概述
---


## API 和端口

Dapr runtime 对外提供两个 API，分别是 Dapr HTTP API 和 Dapr gRPC API。两个 Dapr API 对外暴露的端口，默认是：

- **3500**： HTTP 端口，可以通过命令行参数 `dapr-http-port` 设置
- **50001**： gRPC 端口，可以通过命令行参数 `dapr-grpc-port` 设置

## 发布流程


### HTTP 方式

```plantuml
title Pub-Sub via HTTP
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    producer
]
participant SDK_client [
    =SDK
    ----
    producer
]
end box
participant daprd_client [
    =daprd
    ----
    producer
]
participant message_broker as "Message Broker"

user_code_client -> SDK_client : PublishEvent() 
note left: pubsub_name="name-1"\ntopic="topic-1"\ndata="[...]"\ndata_content_type=""\nmetadata="[...]"
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> message_broker : native protocol (remote call)
|||
message_broker --[#red]> daprd_client :
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```


### gRPC 方式

```plantuml
title Pub-Sub via gRPC
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    producer
]
participant SDK_client [
    =SDK
    ----
    producer
]
end box
participant daprd_client [
    =daprd
    ----
    producer
]
participant message_broker as "Message Broker"

user_code_client -> SDK_client : PublishEvent() 
note left: pubsub_name="name-1"\ntopic="topic-1"\ndata="[...]"\ndata_content_type=""\nmetadata="[...]"
SDK_client -[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001
|||
daprd_client -[#red]> message_broker : native protocol (remote call)
|||
message_broker --[#red]> daprd_client :
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```



