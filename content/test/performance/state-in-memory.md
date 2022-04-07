---
title: "性能测试案例 state in-momery 的实现"
linkTitle: "state in-momery"
weight: 500
date: 2021-05-09
description: >
  性能测试案例 state in-momery 的实现
---



## 测试逻辑

设想中的redis压力测试案例：

```plantuml
title state load test with redis store
hide footbox
skinparam style strictuml

actor test_case as "Test Case"
participant tester as "Tester"
participant fortio as "Fortio"
box "daprd" #LightBlue
participant grpc_api as "gRPC API"
participant redis_component as "Redis State Component"
end box
database redis_server as "Redis server"

test_case -> tester : baseline test
note left: baseline test 
tester -> fortio : exec
fortio -> redis_server: redis native protocol
fortio <-- redis_server
tester <-- fortio
test_case <-- tester

|||
|||
|||

test_case -> tester : dapr test
note left: dapr test 
tester -> fortio : exec
fortio -> grpc_api : gRPC
grpc_api -> redis_component
redis_component -> redis_server: redis native protocol
redis_component <-- redis_server
grpc_api <-- redis_component
fortio <-- grpc_api
tester <-- fortio
test_case <-- tester
```

设想中的 in-memory 压力测试案例：

```plantuml
title state load test with in-memory store
hide footbox
skinparam style strictuml

actor test_case as "Test Case"
participant tester as "Tester"
participant fortio as "Fortio"
box "daprd" #LightBlue
participant grpc_api as "gRPC API"
participant in_momory_component as "In-Memory Component"
end box

test_case -> tester : baseline test
note left: baseline test 
tester -> fortio : exec
fortio -> fortio : do nothing
tester <-- fortio
test_case <-- tester

|||
|||
|||

test_case -> tester : dapr test
note left: dapr test 
tester -> fortio : exec
fortio -> grpc_api : gRPC
grpc_api -> in_momory_component
in_momory_component -> in_momory_component: in-memory operations
grpc_api <-- in_momory_component
fortio <-- grpc_api
tester <-- fortio
test_case <-- tester
```

设想中的 in-memory 压力测试案例：

```plantuml
title state load test with in-memory store
hide footbox
skinparam style strictuml

actor test_case as "Test Case"
participant tester as "Tester"
participant fortio as "Fortio"
box "daprd" #LightBlue
participant grpc_api as "gRPC API"
participant in_momory_component as "In-Memory Component"
end box

test_case -> tester : dapr test
note left: dapr test 
tester -> fortio : exec
fortio -> grpc_api : gRPC
grpc_api -> in_momory_component
in_momory_component -> in_momory_component: in-memory operations
grpc_api <-- in_momory_component
fortio <-- grpc_api
tester <-- fortio
test_case <-- tester
```



