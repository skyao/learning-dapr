---
type: docs
title: "limiter.go的源码学习"
linkTitle: "limiter.go"
weight: 111
date: 2021-02-27
description: >
  rating limiter的代码实现和使用场景
---

Dapr concurrency package中的 limiter.go 文件的源码学习，rating limiter的代码实现和使用场景。

**重点：充分利用 golang chan 的特性**

## 代码实现

### Limiter 结构体定义

```go
// Limiter object
type Limiter struct {
   limit         int
   tickets       chan int
   numInProgress int32
}
```

字段说明：

- limit：最大并发数的限制，这是一个配置项，默认100，初始化后不再修改。
- tickets：用 go 的 chan 来保存和分发 tickets
- numInProgress：当前正在执行中的数量，这是一个实时状态

### 构建Limiter

```go
const (
   // DefaultLimit is the default concurrency limit
   DefaultLimit = 100
)

// NewLimiter allocates a new ConcurrencyLimiter
func NewLimiter(limit int) *Limiter {
   if limit <= 0 {
      limit = DefaultLimit
   }

   // allocate a limiter instance
   c := &Limiter{
      limit:   limit,
      // tickets chan 的 size 设置为 limit
      tickets: make(chan int, limit),
   }

   // allocate the tickets:
   // 开始时先准备和limit数量相当的可用 tickets
   for i := 0; i < c.limit; i++ {
      c.tickets <- i
   }

   return c
}
```

### Limiter的实现

```go
// Execute adds a function to the execution queue.
// if num of go routines allocated by this instance is < limit
// launch a new go routine to execute job
// else wait until a go routine becomes available
func (c *Limiter) Execute(job func(param interface{}), param interface{}) int {
   // 从 chan 中拿一个有效票据
   // 如果当前 chan 中有票据，则说明 go routines 的数量还没有达到 limit 的最大限制，还可以继续启动go routine执行job
   // 如果当前 chan 中没有票据，则说明 go routines 的数量已经达到 limit 的最大限制，需要限速了。execute方法会阻塞在这里，等待有job执行完成释放票据
   ticket := <-c.tickets
   // 拿到之后更新numInProgress，数量加一，要求是原子操作
   atomic.AddInt32(&c.numInProgress, 1)
   // 启动 go routine 执行 job
   go func(param interface{}) {
      // 通过defer来做 job 完成后的清理
      defer func() {
         // 将票据释放给 chan，这样后续的 job 有机会申请到
         c.tickets <- ticket
         // 更新numInProgress，数量减一，要求是原子操作
         atomic.AddInt32(&c.numInProgress, -1)
      }()

      // 执行job
      job(param)
   }(param)
   
   // 返回当前的票据号
   return ticket
}
```

### wait方法

wait方法会阻塞并等待所有的已经通过 execute() 方法拿到票据的 go routine 执行完毕。

```go
// Wait will block all the previously Executed jobs completed running.
//
// IMPORTANT: calling the Wait function while keep calling Execute leads to
//            un-desired race conditions
func (c *Limiter) Wait() {
   // 这是从 chan 中读取所有的票据，只要有任何票据被 job 释放都会去争抢
   // 最后wait()方法获取到所有的票据，其他 job 自然就无法获取票据从而阻塞住所有job的工作
   // 但这并不能保证一定能第一时间抢的到，如果还有其他的 job 也在调用 execute() 方法申请票据，那只有等这个 job 完成工作释放票据时再次争抢
   for i := 0; i < c.limit; i++ {
      <-c.tickets
   }
}
```

## 使用场景

### 并行执行批量操作时限速

在 `pkg/grpc/api.go` 和 `pkg/http/api.go` 的 GetBulkState（）方法中，通过 limiter 来限制批量操作的并发数量：

```go
// 构建limiter，limit参数由 请求参数中的 Parallelism 制定
limiter := concurrency.NewLimiter(int(in.Parallelism))
n := len(reqs)
for i := 0; i < n; i++ {
   fn := func(param interface{}) {
		......
   }
    // 提交 job 给 limiter
   limiter.Execute(fn, &reqs[i])
}

// 等待所有的 job 执行完成
limiter.Wait()
```

在 actor 中也有类似的代码:

```go
limiter := concurrency.NewLimiter(actorMetadata.RemindersMetadata.PartitionCount)
for i := range getRequests {
    fn := func(param interface{}) {
    	......
    }
    limiter.Execute(fn, &bulkResponse[i])
}
limiter.Wait()
```

