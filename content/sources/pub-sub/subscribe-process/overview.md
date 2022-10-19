---
title: "流程概述"
linkTitle: "概述"
weight: 1
date: 2022-04-21
description: >
  Dapr订阅的流程和API概述
---


## API 和端口

订阅流程实际包含三个子流程：

1. 获取应用订阅消息

   daprd 需要获知应用的订阅信息。

   实现中，dapr 会要求应用收集订阅信息并通过指定方式暴露（SDK 可以提供帮助），以便 daprd 可以通过给应用发送请求来获取这些订阅信息。

2. 执行消息订阅

   Daprd 在拿到应用的订阅信息之后，就可以使用底层组件的订阅机制进行消息订阅。

3. 转发消息给应用

   daprd 收到来自底层组件的订阅的消息之后，需要将消息转发给应用。

以上子流程1和3都需要 daprd 主动访问应用，因此 dapr 需要获知应用在哪个端口监听并处理订阅请求，这个信息通过命令行参数 `app-port` 设置。Dapr 的示例中一般喜欢用 3000 端口。

### gRPC API

gRPC API 定义在 `dapr/proto/runtime/v1/appcallback.proto` 文件中的 AppCallback service 中：

```protobuf
service AppCallback {
  // 子流程1:获取应用订阅消息
  rpc ListTopicSubscriptions(google.protobuf.Empty) returns (ListTopicSubscriptionsResponse) {}

  // 子流程3:转发消息给应用
  rpc OnTopicEvent(TopicEventRequest) returns (TopicEventResponse) {}
  ......
}
```

ListTopicSubscriptionsResponse 的定义:

```protobuf
message ListTopicSubscriptionsResponse {
  repeated common.v1.TopicSubscription subscriptions = 1;
}

message TopicSubscription {
  // pubsub的组件名
  string pubsub_name = 1;

  // 要订阅的topic
  string topic = 2;

  // 可选参数，后面展开
  map<string,string> metadata = 3;
  TopicRoutes routes = 5;
  string dead_letter_topic = 6;
}
```

即应用可以有多个消息订阅，每个订阅都必须提供 pubsub_name 和 topic 参数。

TopicEventRequest 的定义：

```protobuf
message TopicEventRequest {
  // 这几个参数先忽略
  string id = 1;
  string source = 2;
  string type = 3;
  string spec_version = 4;
  string path = 9;

  // 事件的基本信息
  string data_content_type = 5;
  bytes data = 7;
  string topic = 6;
  string pubsub_name = 8;
}
```

### HTTP API



## 发布流程


### HTTP 协议

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



