---
title: "资源绑定API的go client定义"
linkTitle: "go client定义"
weight: 724
date: 2021-01-31
description: >
  Dapr的资源绑定API的go client定义
---

### DaprClient

/pkg/proto/runtime/v1/dapr.pb.go

```go
// DaprClient is the client API for Dapr service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type DaprClient interface {
	// Invokes binding data to specific output bindings
	InvokeBinding(ctx context.Context, in *InvokeBindingRequest, opts ...grpc.CallOption) (*InvokeBindingResponse, error)
	......
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

func (c *daprClient) InvokeBinding(ctx context.Context, in *InvokeBindingRequest, opts ...grpc.CallOption) (*InvokeBindingResponse, error) {
	out := new(InvokeBindingResponse)
  // 调用固定的grpc方法 `/dapr.proto.runtime.v1.Dapr/InvokeBinding`
	err := c.cc.Invoke(ctx, "/dapr.proto.runtime.v1.Dapr/InvokeBinding", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```



### AppCallbackClient

/pkg/proto/runtime/v1/appcallback.pb.go

```go
// AppCallbackClient is the client API for AppCallback service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type AppCallbackClient interface {
  // Lists all input bindings subscribed by this app.
	ListInputBindings(ctx context.Context, in *empty.Empty, opts ...grpc.CallOption) (*ListInputBindingsResponse, error)
	// Listens events from the input bindings
	//
	// User application can save the states or send the events to the output
	// bindings optionally by returning BindingEventResponse.
	OnBindingEvent(ctx context.Context, in *BindingEventRequest, opts ...grpc.CallOption) (*BindingEventResponse, error)
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

