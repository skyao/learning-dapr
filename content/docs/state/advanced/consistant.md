---
title: "状态管理高级特性之一致性"
linkTitle: "一致性"
weight: 843
date: 2021-01-31
description: >
  Dapr状态管理高级特性之一致性
---


## 设计分析

dapr state 目前对操作的一致性要求有两个： strong 和 eventual。

```go
const (
	Strong     = "strong"
	Eventual   = "eventual"
)
```

eventual 就简单了，每个写操作都只需要简单的执行即可，后续的同步等操作由底层实现自行保证。

FirstWrite （First Write Win模式）复杂一些，当有多个操作进行并发写时，只有第一个能成功。因此，必须有机制能够在执行写操作时判断从上次读到这次写，期间 state 数据没有被修改。也就是需要实现 CAS操作：CAS  = Compare And Set。

Dapr state 的设计是引入一个名为 ETag 的机制：

- ETag 是一个整型，每个状态都会关联一个 ETag
- 每次创建或修改 state 时，ETag都会递增
- 进行写操作时：先读取现有state，拿到当前的ETag；在提交写操作时，传入之前的ETag。底层 state store的实现应该在执行写操作之前检查ETag是否匹配。

具体到各个操作：

1. Save state
   - grpc API：在请求的SaveStateRequest中通过 etag 字段提供
   - HTTP API：在请求的json内容中通过etag字段提供
2. Get state
   - grpc API：在应答的 GetStateResponse 中通过 etag 字段提供
   - HTTP API：在应答的 ETag header中提供
3. Get Bulk
   - grpc API：在应答的 GetBulkStateResponse 中通过 etag 字段提供
   - HTTP API：在应答的json中通过 etag 字段提供
4. Delete State
   - grpc API：在请求的 DeleteStateRequest 中通过 etag 字段提供
   - HTTP API：通过请求的 If-Match header提供




## 实现分析

### Redis实现

redis 为了实现 state 要求的 etag，就必须在常规的key/value存储模型上增加 key/etag 的存储，实现方式就是 key / map as value，将一个 map 作为value（刚好redis本身也支持map结构）。然后在map中存储 data / version 等多个信息：

- key=version，存储ETag需要的version
- key=data，存储state的实际数据

读取state的时候将整个map as value读取，然后分别取data和version即可。

但写操作会比较麻烦， redis 本身不直接提供对多个字段的原子操作方式，因此在save和delete操作时需要通过LUA脚本来完成。

- concurrency 设置为 first-write ：需要通过 etag 实现 CAS （Compare And Set）
- concurrency 设置为 last-write ：忽略 etag，即使请求设置了也要重置