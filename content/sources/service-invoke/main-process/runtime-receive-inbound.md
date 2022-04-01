---
title: "Dapr Runtime接收服务调用的inbound请求"
linkTitle: "Runtime接收inbound请求"
weight: 60
date: 2021-01-31
description: >
  Dapr Runtime通过gRPC internal API接收来自客户端Dapr Runtime的inbound请求
---

Dapr runtime 之间相互通讯走的是 gRPC internal API，这个 API 也只支持 gRPC 协议。

```plantuml
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

daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port\n/dapr.proto.internals.v1.ServiceInvocation/CallLocal
daprd_server -> daprd_server : interceptor
daprd_server -[#blue]>  : appChannel.InvokeMethod()
```

### 接收请求

Runtime 初始化时，在注册 gRPC 服务时绑定了 gPRC Internal API 实现和 CallLocal gRPC 方法。对于访问 `dapr.proto.internals.v1.ServiceInvocation` 服务的 `CallLocal` 方法的 gRPC 请求，会将请求转给 `_ServiceInvocation_CallLocal_Handler` 处理:

```go
func _ServiceInvocation_CallLocal_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	......
	if interceptor == nil {
		return srv.(ServiceInvocationServer).CallLocal(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/dapr.proto.internals.v1.ServiceInvocation/CallLocal",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        // 这里调用的 srv 即 gRPC api 实现
		return srv.(ServiceInvocationServer).CallLocal(ctx, req.(*InternalInvokeRequest))  
	}
	return interceptor(ctx, in, info, handler)
}
```

最后进入 CallLocal() 方法进行处理。

> 备注：初始化的细节，请见前面章节 "Runtime初始化"

期间会有一个 interceptor 的处理流程，细节后面展开。

### 转发请求

当 internal invoke 的 gRPC 请求进来后，就会进入 `pkc/grpc/api.go` 中的 CallLocal 方法：

```go
func (a *api) CallLocal(ctx context.Context, in *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error) {
	// 1. 构造请求
	req, err := invokev1.InternalInvokeRequest(in)
  if a.accessControlList != nil {
		......
	}
	// 2. 通过 appChannel 向应用发出请求
	resp, err := a.appChannel.InvokeMethod(ctx, req)
    // 3. 处理应答
	return resp.Proto(), err
}
```

处理方式很清晰，基本上就是将请求通过 app channel 转发。Runtime 本身并没有什么额外的处理逻辑。InternalInvokeRequest() 只是简单处理一下参数：

```go
// InternalInvokeRequest creates InvokeMethodRequest object from InternalInvokeRequest pb object.
func InternalInvokeRequest(pb *internalv1pb.InternalInvokeRequest) (*InvokeMethodRequest, error) {
	req := &InvokeMethodRequest{r: pb}
	if pb.Message == nil {
		return nil, errors.New("Message field is nil")
	}

	return req, nil
}
```

### 访问控制

期间会有一个 access control （访问控制）的逻辑:

```go
	if a.accessControlList != nil {
		// An access control policy has been specified for the app. Apply the policies.
		operation := req.Message().Method
		var httpVerb commonv1pb.HTTPExtension_Verb
		// Get the http verb in case the application protocol is http
		if a.appProtocol == config.HTTPProtocol && req.Metadata() != nil && len(req.Metadata()) > 0 {
			httpExt := req.Message().GetHttpExtension()
			if httpExt != nil {
				httpVerb = httpExt.GetVerb()
			}
		}
		callAllowed, errMsg := acl.ApplyAccessControlPolicies(ctx, operation, httpVerb, a.appProtocol, a.accessControlList)

		if !callAllowed {
			return nil, status.Errorf(codes.PermissionDenied, errMsg)
		}
	}
```

细节后面展开。
