---
title: "流程概述"
linkTitle: "概述"
weight: 1
date: 2021-01-29
description: >
  Dapr服务调用的流程和API概述
---


## API 和端口

Dapr runtime 对外提供两个 API，分别是 Dapr HTTP API 和 Dapr gRPC API。另外两个 dapr runtime 之间的通讯 (Dapr internal API) 固定用 gRPC 协议。

两个 Dapr API 对外暴露的端口，默认是：

- **3500**： HTTP 端口，可以通过命令行参数 `dapr-http-port` 设置
- **50001**： gRPC 端口，可以通过命令行参数 `dapr-grpc-port` 设置

Dapr internal API 是内部端口，比较特殊，没有固定的默认值，而是取任意随机可用端口。也可以通过命令行参数 `dapr-internal-grpc-port` 设置。

为了向服务器端的应用发送请求，dapr 需要获知应用在哪个端口监听并处理请求，这个信息通过命令行参数 `app-port` 设置。Dapr 的示例中一般喜欢用 3000 端口。

## 调用流程


### HTTP 方式

```plantuml
title Service Invoke via HTTP
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    client
]
participant SDK_client [
    =SDK
    ----
    client
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant user_code_server [
    =App-2
    ----
    server
]
end box

user_code_client -> SDK_client : Invoke\nService() 
note left: appId="app-2"\nmethodName="method-1"
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port
|||
daprd_server -[#blue]> user_code_server :  http (localhost)
note right: HTTP endpoint "method-1" @ 3000

daprd_server <[#blue]-- user_code_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```


### gRPC 方式

```plantuml
title Service Invoke via gRPC
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    client
]
participant SDK_client [
    =SDK
    ----
    client
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant SDK_server [
    =SDK
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]
end box
user_code_server -> SDK_server: AddServiceInvocationHandler("method-1")
SDK_server -> SDK_server: save handler in invokeHandlers["method-1"]
SDK_server --> user_code_server
user_code_client -> SDK_client : Invoke\nService() 
note left: appId="app-2"\nmethodName="method-1"
SDK_client -[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001\n/dapr.proto.runtime.v1.Dapr/InvokeService
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ random free port\n/dapr.proto.internals.v1.ServiceInvocation/CallLocal
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001\n/dapr.proto.runtime.v1.AppCallback/OnInvoke
SDK_server -> SDK_server: get handler by invokeHandlers["method-1"]
SDK_server -> user_code_server : invoke handler of "method-1"

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```

### gRPC  proxying 方式

```plantuml
title Service Invoke via gRPC proxying
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    client
]
participant SDK_client [
    =SDK
    ----
    client
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant SDK_server [
    =gRPC
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]
end box
user_code_server -> SDK_server
SDK_server --> user_code_server
user_code_client -[#blue]> daprd_client : gRPC (localhost)
note right: gRPC\n/user.services.ServiceName/Method-1
|||
daprd_client -[#red]> daprd_server : gRPC proxy (remote call)
note right: gRPC\n/user.services.ServiceName/Method-1
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: gRPC\n/user.services.ServiceName/Method-1
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```



