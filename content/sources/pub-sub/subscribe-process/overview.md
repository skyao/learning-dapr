---
title: "流程概述"
linkTitle: "概述"
weight: 1
date: 2022-04-21
description: >
  Dapr订阅的流程和API概述
---


## API 和端口

为了向服务器端的应用发送订阅请求，dapr 需要获知应用在哪个端口监听并处理订阅请求，这个信息通过命令行参数 `app-port` 设置。Dapr 的示例中一般喜欢用 3000 端口。

## 发布流程


### HTTP 方式

```plantuml
title Subscribe via http
hide footbox
skinparam style strictuml
box "App-1"
participant user_code [
    =App-1
    ----
    producer
]
participant SDK [
    =SDK
    ----
    producer
]
end box
participant daprd [
    =daprd
    ----
    producer
]
participant message_broker as "Message Broker"

SDK -> user_code: collection subscribe
user_code --> SDK

daprd -[#blue]> SDK : http
note left: appChannel.InvokeMethod("dapr/subscribe")
SDK --[#blue]> daprd : 

daprd -[#red]> message_broker : subscribe topics
message_broker --[#red]> daprd

|||
|||
|||
|||

message_broker -[#red]> daprd: event
daprd -[#blue]> SDK : http
note left: appChannel.InvokeMethod("/{route}")
SDK -> user_code : 
user_code --> SDK
SDK --[#blue]> daprd
|||
```


### gRPC 方式

```plantuml
title Subscribe via gRPC
hide footbox
skinparam style strictuml
box "App-1"
participant user_code [
    =App-1
    ----
    producer
]
participant SDK [
    =SDK
    ----
    producer
]
end box
participant daprd [
    =daprd
    ----
    producer
]
participant message_broker as "Message Broker"

SDK -> user_code: collection subscribe
user_code --> SDK

daprd -[#blue]> SDK : gRPC
note left: appChannel.ListTopicSubscriptions()
SDK --[#blue]> daprd : 

daprd -[#red]> message_broker : subscribe topics
message_broker --[#red]> daprd

|||
|||
|||
|||

message_broker -[#red]> daprd: event
daprd -[#blue]> SDK : gRPC
note left: appChannel.OnTopicEvent()
SDK -> user_code : 
user_code --> SDK
SDK --[#blue]> daprd
|||
```



