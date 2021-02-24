---
title: "服务调用的go client定义"
linkTitle: "go client定义"
weight: 524
date: 2021-01-29
description: >
  Dapr服务调用的go client定义
---

### DaprClient

https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/pkg/proto/runtime/v1/dapr.pb.go

```go
// DaprClient is the client API for Dapr service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type DaprClient interface {
	// Invokes a method on a remote Dapr app.
	InvokeService(ctx context.Context, in *InvokeServiceRequest, opts ...grpc.CallOption) (*v1.InvokeResponse, error)
	......s
}
```
DaprClient 的实现：

```go
type daprClient struct {
	cc *grpc.ClientConn
}

func NewDaprClient(cc *grpc.ClientConn) DaprClient {
	return &daprClient{cc}
}

func (c *daprClient) InvokeService(ctx context.Context, in *InvokeServiceRequest, opts ...grpc.CallOption) (*v1.InvokeResponse, error) {
	out := new(v1.InvokeResponse)
	err := c.cc.Invoke(ctx, "/dapr.proto.runtime.v1.Dapr/InvokeService", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```



### AppCallbackClient

https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/pkg/proto/runtime/v1/appcallback.pb.go

```go
// AppCallbackClient is the client API for AppCallback service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type AppCallbackClient interface {
	// Invokes service method with InvokeRequest.
	OnInvoke(ctx context.Context, in *v1.InvokeRequest, opts ...grpc.CallOption) (*v1.InvokeResponse, error)
	......
}
```

AppCallbackClient 的实现：

```go
type appCallbackClient struct {
	cc *grpc.ClientConn
}

func NewAppCallbackClient(cc *grpc.ClientConn) AppCallbackClient {
	return &appCallbackClient{cc}
}

func (c *appCallbackClient) OnInvoke(ctx context.Context, in *v1.InvokeRequest, opts ...grpc.CallOption) (*v1.InvokeResponse, error) {
	out := new(v1.InvokeResponse)
	err := c.cc.Invoke(ctx, "/dapr.proto.runtime.v1.AppCallback/OnInvoke", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

