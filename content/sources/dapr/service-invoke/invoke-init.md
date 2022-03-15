---
type: docs
title: "服务调用的初始化"
linkTitle: "初始化"
weight: 220
date: 2021-01-31
description: >
  Dapr服务调用的初始化流程和源码分析
---

## gRPC

### 注册 Dapr Service 到 gRPC 服务器

为了让 dapr runtime （daprd）的 gRPC 服务器能挂载 Dapr Service，需要在 dapr runtime 启动做初始化时，将 Dapr_ServiceDesc 注册上去：

实现代码在 `pkg/proto/runtime/v1/dapr_grpc.pb.go`:

```
func RegisterDaprServer(s grpc.ServiceRegistrar, srv DaprServer) {
	s.RegisterService(&Dapr_ServiceDesc, srv)
}
```

### Dapr_ServiceDesc 定义

在文件 `pkg/proto/runtime/v1/dapr_grpc.pb.go` 中有 Dapr Service 的 grpc 服务定义，这是 protoc 生成的 gRPC 代码。

Dapr_ServiceDesc中有 Dapr Service 各个方法的定义，和服务调用相关的是 `InvokeService` 方法：

```go
var Dapr_ServiceDesc = grpc.ServiceDesc{
	ServiceName: "dapr.proto.runtime.v1.Dapr",
	HandlerType: (*DaprServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "InvokeService",				# 注册方法名
			Handler:    _Dapr_InvokeService_Handler,	# 关联实现的 Handler
		},
        ......
        },
	},
	Metadata: "dapr/proto/runtime/v1/dapr.proto",
}
```

这一段是告诉 gRPC server： 如果收到访问 `dapr.proto.runtime.v1.Dapr` 服务的 `InvokeService` 方法的  gRPC 请求，请把请求转给 `_Dapr_InvokeService_Handler` 处理。

而 `InvokeService` 方法相关联的 handler 方法 `_Dapr_InvokeService_Handler `的实现代码是：

```go
func _Dapr_InvokeService_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(InvokeServiceRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(DaprServer).InvokeService(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/dapr.proto.runtime.v1.Dapr/InvokeService",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(DaprServer).InvokeService(ctx, req.(*InvokeServiceRequest))
	}
	return interceptor(ctx, in, info, handler)
}
```





## HTTP



