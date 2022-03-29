---
type: docs
title: "服务调用的runtime接收请求"
linkTitle: "Runtime 接收请求"
weight: 40
date: 2021-01-31
description: >
  Dapr服务调用的客户端sdk封装

---

Dapr runtime 有两种方式接收来自客户端发起的服务调用的请求：gRPC API 和 HTTP API。

## gRPC API

Runtime 初始化时，在注册 gRPC 服务时绑定了 gPRC API 实现和 InvokeService gRPC 方法。

当 service invoke 的 gRPC 请求进来后，就会进入 `pkc/grpc/api.go` 中的 InvokeService 方法：

```go
func (a *api) InvokeService(ctx context.Context, in *runtimev1pb.InvokeServiceRequest) (*commonv1pb.InvokeResponse, error) {
	......
	resp, err := a.directMessaging.Invoke(ctx, in.Id, req)
	......
	return resp.Message(), respError
}
```

gRPC API 的实现特别简单，除了基本的请求/应答参数处理之外，就是将转发请求的事情交给了 directMessaging。

## HTTP API

Runtime 初始化时，在注册 HTTP 服务时绑定了 handler 实现和 URL 路由。

当 service invoke 的 HTTP 请求进来后，就会被 fasthttp 路由到 Handler 即 HTTP API 实现的 onDirectMessage() 方法中进行处理。

onDirectMessage 的实现代码在文件 `pkg/http/api.go`, 示意如下：

```go
func (a *api) onDirectMessage(reqCtx *fasthttp.RequestCtx) {
	......
  req := invokev1.NewInvokeMethodRequest(...)
	resp, err := a.directMessaging.Invoke(reqCtx, targetID, req)
	......
}
```

备注： HTTP API 的这个 onDirectMessage() 方法取名不对，应该效仿 gRPC API，取名为 InvokeService(). 理由是：这是暴露给外部调用的方法，取名应该表现出它对外暴露的功能，即InvokeService。而不应该暴露内部的实现是调用 directMessaging。

HTTP API 的实现也简单，同样，除了基本的请求/应答参数处理之外，就是将转发请求的事情交给了 directMessaging。

