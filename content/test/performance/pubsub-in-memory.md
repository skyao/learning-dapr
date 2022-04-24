---
title: "性能测试案例 pubsub in-momery 的实现"
linkTitle: "pubsub in-momery"
weight: 400
date: 2021-05-09
description: >
  性能测试案例 pubsub in-momery 的实现
---



## 测试逻辑

和 state in-momery 类似。

## fortio命令

对于 baseline test 中的 no-op，fortio 的命令为：

```bash
./fortio load -json result.json -qps 1 -c 1 -t 1m -payload-size 0 -grpc -dapr capability=pubsub,target=noop http://localhost:50001/
```

对于 dapr test 中的 state get 请求，fortio 的命令为：

```bash
./fortio load -json result.json -qps 1 -c 1 -t 1m -payload-size 100 -grpc --dapr capability=pubsub,target=dapr,method=publish,store=inmemorypubsub,topic=topic123,contenttype=text/plain http://127.0.0.1:50001
```

## perf test case 的请求

