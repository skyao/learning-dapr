---
type: docs
title: "Dapr Runtime接收服务调用的outbound请求"
linkTitle: "Runtime接收outbound请求"
weight: 40
date: 2021-01-31
description: >
  Dapr Runtime通过gRPC API 和 HTTP API接收来自应用的outbound请求
---

Dapr runtime 有两种方式接收来自客户端发起的服务调用的 outbound 请求：gRPC API 和 HTTP API。在接收到请求之后，dapr runtime 会将 outbound 请求转发给目标服务的 dapr runtime。

```plantuml
title Daprd Receive inbound Request
hide footbox
skinparam style strictuml

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

 -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500 \n/v1.0/invoke/app-2/method/method-1
 -[#blue]> daprd_client : gRPC (localhost)
note right: GRPC API @ 50001\n/dapr.proto.runtime.v1.Dapr/InvokeService
|||
daprd_client -> daprd_client: name resolution
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
```



## HTTP API

Runtime 初始化时，在注册 HTTP 服务时绑定了 handler 实现和 URL 路由:

```go
func (a *api) constructDirectMessagingEndpoints() []Endpoint {
	return []Endpoint{
		{
			Methods:           []string{router.MethodWild},
			Route:             "invoke/{id}/method/{method:*}",
			Alias:             "{method:*}",
			Version:           apiVersionV1,
			KeepParamUnescape: true,
			Handler:           a.onDirectMessage,
		},
	}
}
```

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



## Name Resolution

TBD
