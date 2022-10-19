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

### gRPC API

gRPC API 定义在 `dapr/proto/runtime/v1/dapr.proto` 文件中的 Dapr service 中：

```protobuf
service Dapr {
  // Publishes events to the specific topic.
  rpc PublishEvent(PublishEventRequest) returns (google.protobuf.Empty) {}
  ......
}

// PublishEventRequest is the message to publish event data to pubsub topic
message PublishEventRequest {
  // The name of the pubsub component
  string pubsub_name = 1;

  // The pubsub topic
  string topic = 2;

  // The data which will be published to topic.
  bytes data = 3;

  // The content type for the data (optional).
  string data_content_type = 4;

  // The metadata passing to pub components
  //
  // metadata property:
  // - key : the key of the message.
  map<string, string> metadata = 5;
}
```

主要的参数是：

- pubsub_name：dapr pubsub component的名字
- topic：发布消息的目标topic
- data：消息的数据

可选参数有：

- data_content_type：消息数据的内容类型
- metadata：可选的元数据信息，用于扩展

### HTTP API

HTTP API 没有明确的单独定义，不过可以从代码中获知。在 `pkg/http/api.go` 中，构建用于 publish 的 endpoint 的代码如下：

```go
func (a *api) constructPubSubEndpoints() []Endpoint {
	return []Endpoint{
		{
      // 发送 POST 或者 PUT 请求
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
      // 到这个 URL
			Route:   "publish/{pubsubname}/{topic:*}",
			Version: apiVersionV1,
			Handler: a.onPublish,
		},
	}
}
```

因此，用于 publish 的 daprd URL 类似于 `http://localhost:3500/v1.0/publish/pubsubname1/topic1`。

处理请求的 handler 方法 a.onPublish() 中读取参数的代码如下（忽略其他细节）：

```go
const (
  pubsubnameparam          = "pubsubname"
）

// 从 url 中读取 pubsubname
pubsubName := reqCtx.UserValue(pubsubnameparam).(string)
// 从 url 中读取 topic
topic := reqCtx.UserValue(topicParam).(string)
// 从 HTTP body 
body := reqCtx.PostBody()
// 从 HTTP 的 Content-Type header 中读取 data_content_type
contentType := string(reqCtx.Request.Header.Peek("Content-Type"))
  
// 从 HTTP URL query 中读取 metadata
metadata := getMetadataFromRequest(reqCtx)
```

Metadata 的读取要稍微复杂一些，需要读取所有的 url query 参数，然后根据 key 的前缀判断是不是 metadata： 

```go
const (
	metadataPrefix        = "metadata."
)

func getMetadataFromRequest(reqCtx *fasthttp.RequestCtx) map[string]string {
	metadata := map[string]string{}
  // 游历所有的 url query 参数
	reqCtx.QueryArgs().VisitAll(func(key []byte, value []byte) {
		queryKey := string(key)
    // 如果 query 参数的 key 以 "metadata." 开头，就视为一个 metadata 的key
		if strings.HasPrefix(queryKey, metadataPrefix) {
      // key 的 前缀 "metadata." 要去掉
			k := strings.TrimPrefix(queryKey, metadataPrefix)
			metadata[k] = string(value)
		}
	})

	return metadata
}
```

总结：用于 publish 的完整的 daprd URL 类似于 `http://localhost:3500/v1.0/publish/pubsubname1/topic1?metadata.k1=v1&metadata.k2=v2&metadata.k3=v3`。消息内容通过 HTTP body 传递，另外可以通过 Content-Type header 传递消息内容类型参数。



## 发布流程

### gRPC 协议

默认情况下使用 gRPC 协议进行消息发布，daprd 在默认的 50001 端口，通过注册的 dapr service 的 PublishEvent() 方法接收来自客户端通过 dapr SDK 发出的 gRPC 请求，之后根据具体的组件实现，对底层实际使用的消息中间件发布事件。流程大体如下：

```plantuml
title Pub-Sub via gRPC Protocol
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =User Code
    ----
    producer
]
participant SDK_client [
    =Dapr SDK
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
note right: PublishEvent() @ Dapr service
SDK_client -[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001
|||
daprd_client -[#red]> message_broker : native protocol (remote call)
|||
message_broker --[#red]> daprd_client :
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```



### HTTP 协议

HTTP协议类似，daprd 在默认的 3500 端口，通过前面所述的URL接收客户端通过 dapr SDK 发出的 HTTP 请求。流程大体如下：

```plantuml
title Pub-Sub via HTTP Protocol
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =User Code
    ----
    producer
]
participant SDK_client [
    =Dapr SDK
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
note right: POST http://localhost:3500/v1.0/publish/pubsubname1/topic1?\nmetadata.k1=v1&metadata.k2=v2&metadata.k3=v3
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> message_broker : native protocol (remote call)
|||
message_broker --[#red]> daprd_client :
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```


### 
