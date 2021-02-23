---
type: blog
date: 2019-07-12
title: "用于应用级追踪组件的追踪API和接口"
linkTitle: "用于应用级追踪组件的追踪API和接口"
description: >
  用于应用级追踪组件的追踪API和接口
---

用于应用级追踪组件的追踪API和接口

## Proposal信息

[[Proposal] Tracing API and Interface for application level tracing components #100](https://github.com/dapr/dapr/issues/100)

Goal: allow users to leverage distributed tracing from their code, regardless of whether the request came in through Dapr.

> 目标：允许用户从他们的代码中利用分布式追踪，无论请求是否通过Dapr进来。

Here's a very initial rough draft to iterate on:

> 这是一个非常初步的草稿，可以反复推敲:

```go
type Trace interface {
	TraceOperation(ctx context.Context, req TraceOperationRequest) error
}

type TraceOperationRequest struct {
	Operation string
	Data      interface{}
}
```

### 提案讨论

How about exposing opentracing-compatible API in the dapr sidecar? It would be great if it would support automated telemetry collectors like Application Insights - which are being ported to opentracing if I understand correctly.

> 在dapr sidecar中暴露与opentracing兼容的API怎么样？如果它能支持像Application Insights这样的自动遥测收集器就更好了--如果我没理解错的话，它正在被移植到opentracing上。

[@mkosieradzki](https://github.com/mkosieradzki) that's a great idea.

We have a plan to make telemetry collectors [Dapr components](https://github.com/dapr/components-contrib] for that purpose.

> @mkosieradzki 这是个好主意。
>
> 我们有一个计划，为这个目的制作遥测采集器Dapr组件

Is this the same as this?
https://github.com/dapr/components-contrib/tree/master/exporters

Not really. What I mean is to expose unauthenticated endpoint in sidecar to which app can send it's internal telemetry, which dapr will forward using the exporter you mention.

> 并非如此。我的意思是在sidecar中暴露出未经认证的端点，应用程序可以向其发送内部遥测数据，dapr将使用你提到的exporter转发。

I like the sidecar idea. Given the communication within a pod between it's containers transpires over the localhost interface, this is by far the quickest way to ingest telemetry via an unsecured endpoint.

> 我喜欢sidecar的想法。鉴于pod内容器之间的通信是通过localhost接口进行的，这是迄今为止通过不安全的端点摄取遥测数据的最快方式。

Yes. Now that we have separated exporters into the components-contrib repo, we are prime to start designing the app facing API.

> 是的。现在我们已经将 exporter 分离到了 components-contrib repo中，我们可以开始设计面向应用的API了。

Implementing collector is not easy, which can be conflicted with opentelemetry collector effort. Instead, we need to provide a way to correlate with user app tracing with dapr sidecar tracing.

> 实现收集器并不容易，这可能与opentelemetry收集器的努力相冲突。相反，我们需要提供一种与用户应用追踪与dapr sidecar追踪相关联的方法。