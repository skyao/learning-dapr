---
type: docs
title: "服务调用的源码概述"
linkTitle: "概述"
weight: 1
date: 2021-01-29
description: >
  Dapr 服务调用构建块的流程和API
---


## API 和端口

Dapr runtime 对外提供两个 API，分别是 Dapr HTTP API 和 Dapr gRPC API。另外两个 dapr runtime 之间的通讯 (Dapr internal API) 固定用 gRPC 协议。

两个 Dapr API 对外暴露的端口，默认是：

- **3500**： HTTP 端口
- **50001**： gRPC 端口

Dapr internal API 是内部端口，比较特殊，没有固定的默认值，而是取任意随机可用端口

## 调用流程

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

user_code_client -> SDK_client : Invoke\nService() 
SDK_client -[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ random free port
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```



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

user_code_client -> SDK_client : Invoke\nService() 
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```



