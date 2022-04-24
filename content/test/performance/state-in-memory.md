---
title: "性能测试案例 state in-momery 的实现"
linkTitle: "state in-momery"
weight: 300
date: 2021-05-09
description: >
  性能测试案例 state in-momery 的实现
---



## 测试逻辑

完整的端到端的 state 压力测试应该是这样的，以 redis 为例：

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

但目前 dapr 仓库中的 perf test 的测试目标都是 dapr runtime，也就是不包括 dapr sdk 和 dapr components，因此，需要一个 in-memory 的 state 组件来进行压力测试以排除 redis 等外部组件的干扰，流程大体是这样：

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

in-memory 的 state 组件的性能消耗可以视为0，因此将访问 redis 的远程开销在 baseline test 和 dapr test 中同时去除之后，得到的新的 baseline test 和 dapr test 如下图：

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



## fortio命令

对于 baseline test 中的 no-op，fortio 的命令为：

```bash
./fortio load -json result.json -qps 1 -c 1 -t 1m -payload-size 0 -grpc -dapr capability=state,target=noop http://localhost:50001/
```

对于 dapr test 中的 state get 请求，fortio 的命令为：

```bash
 ./fortio load -json result.json -qps 1 -c 1 -t 1m -payload-size 0 -grpc --dapr capability=state,target=dapr,method=get,store=inmemorystate,key=abc123 http://127.0.0.1:50001
```

## perf test case 的请求

