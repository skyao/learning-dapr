---
type: docs
title: "服务调用的源码概述"
linkTitle: "概述"
weight: 210
date: 2021-01-29
description: >
  Dapr 服务调用构建块的源码概述
---



服务调用/Service Invoke 构建块的源码



## 流程



```plantuml

actor       		user_code_client
participant 		SDK_client
participant 		daprd_client
participant 		daprd_server
participant 		SDK_server
actor       		user_code_server

user_code_client -> SDK_client : InvokeService() 
SDK_client -> daprd_client : InvokeService() 
```



@startuml
participant Participant as Foo
actor       Actor       as Foo1
boundary    Boundary    as Foo2
control     Control     as Foo3
entity      Entity      as Foo4
database    Database    as Foo5
collections Collections as Foo6
queue       Queue       as Foo7
Foo -> Foo1 : To actor 
Foo -> Foo2 : To boundary
Foo -> Foo3 : To control
Foo -> Foo4 : To entity
Foo -> Foo5 : To database
Foo -> Foo6 : To collections
Foo -> Foo7: To queue
@enduml