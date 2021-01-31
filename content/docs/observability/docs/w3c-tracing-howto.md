---
title: "如何在Dapr中使用的W3C跟踪上下文"
linkTitle: "如何在Dapr中使用的W3C跟踪上下文"
weight: 916
date: 2021-01-31
description: >
  如何在Dapr中使用的W3C跟踪上下文
---


> 内容节选自：https://docs.dapr.io/developing-applications/building-blocks/observability/w3c-tracing/w3c-tracing-howto/

使用Dapr的W3C追踪标准。

## 如何从响应中检索跟踪上下文。

> 注意：在Dapr SDK中没有暴露的帮助方法来传播和检索跟踪上下文。你需要使用http/gRPC客户端通过http头和gRPC元数据来传播和检索跟踪头。

### 在Go中检索跟踪上下文

#### 对于HTTP调用

OpenCensus Go SDK提供ochttp包，它提供了从http响应中获取跟踪上下文的方法。

要从HTTP响应中获取跟踪上下文，你可以这样：

```go
f := tracecontext.HTTPFormat{}
sc, ok := f.SpanContextFromRequest(req)
```

#### 对于gRPC调用

要在gRPC调用返回时检索跟踪上下文头，可以将响应头引用作为gRPC调用选项传递，该选项包含响应头。

```go
var responseHeader metadata.MD

// Call the InvokeService with call option
// grpc.Header(&responseHeader)

client.InvokeService(ctx, &pb.InvokeServiceRequest{
		Id: "client",
		Message: &commonv1pb.InvokeRequest{
			Method:      "MyMethod",
			ContentType: "text/plain; charset=UTF-8",
			Data:        &any.Any{Value: []byte("Hello")},
		},
	},
	grpc.Header(&responseHeader))
```

## 如何在请求中传播跟踪上下文

> 注意：在Dapr SDK中没有暴露的帮助方法来传播和检索跟踪上下文。你需要使用http/gRPC客户端通过http头和gRPC元数据来传播和检索跟踪头。

### 在Go中传递跟踪上下文

#### 对于HTTP调用

OpenCensus Go SDK提供ochttp包，提供在http请求中附加跟踪上下文的方法。

```go
f := tracecontext.HTTPFormat{}
req, _ := http.NewRequest("GET", "http://localhost:3500/v1.0/invoke/mathService/method/api/v1/add", nil)

traceContext := span.SpanContext()
f.SpanContextToRequest(traceContext, req)
```

#### 对于gRPC调用

```go
traceContext := span.SpanContext()
traceContextBinary := propagation.Binary(traceContext)
```

然后你可以通过gRPC元数据用 `grpc-trace-bin` 头传递跟踪上下文。

```go
ctx = metadata.AppendToOutgoingContext(ctx, "grpc-trace-bin", string(traceContextBinary))
```

...

