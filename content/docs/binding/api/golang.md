---
title: "资源绑定API的Golang生成代码"
linkTitle: "Golang生成代码"
weight: 723
date: 2021-01-31
description: >
  Dapr的资源绑定API的Golang生成代码
---

从proto api定义文件生成的golang代码，被存放在dapr项目的 ` pkg/proto/` 目录下。

### grpc服务定义

DaprServer 是 dapr 服务的服务器端API定义，包含 InvokeBinding 方法：

```go
// DaprServer is the server API for Dapr service.
type DaprServer interface {
	// Invokes binding data to specific output bindings
	InvokeBinding(context.Context, *InvokeBindingRequest) (*InvokeBindingResponse, error)
   ......
}
```

AppCallbackServer 是 AppCallback 服务的服务器端API定义，包含 ListInputBindings 方法和 OnBindingEvent 方法：

```go
// AppCallbackServer is the server API for AppCallback service.
type AppCallbackServer interface {
	// Lists all input bindings subscribed by this app.
	ListInputBindings(context.Context, *empty.Empty) (*ListInputBindingsResponse, error)
	// Listens events from the input bindings
	//
	// User application can save the states or send the events to the output
	// bindings optionally by returning BindingEventResponse.
	OnBindingEvent(context.Context, *BindingEventRequest) (*BindingEventResponse, error)
	......
}
```

### InvokeBindingRequest的定义

pkg/proto/runtime/v1/dapr.pb.go：

```go
// InvokeBindingResponse is the message returned from an output binding invocation
type InvokeBindingResponse struct {
	// The data which will be sent to output binding.
	Data []byte `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
	// The metadata returned from an external system
	Metadata             map[string]string `protobuf:"bytes,2,rep,name=metadata,proto3" json:"metadata,omitempty" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

### InvokeBindingResponse的定义

pkg/proto/runtime/v1/dapr.pb.go：

```go
// InvokeBindingResponse is the message returned from an output binding invocation
type InvokeBindingResponse struct {
	// The data which will be sent to output binding.
	Data []byte `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
	// The metadata returned from an external system
	Metadata             map[string]string `protobuf:"bytes,2,rep,name=metadata,proto3" json:"metadata,omitempty" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

### ListInputBindingsResponse的定义

pkg/proto/runtime/v1/appcallback.pb.go：

```go
// ListInputBindingsResponse is the message including the list of input bindings.
type ListInputBindingsResponse struct {
	// The list of input bindings.
	Bindings             []string `protobuf:"bytes,1,rep,name=bindings,proto3" json:"bindings,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```

### BindingEventRequest的定义

pkg/proto/runtime/v1/appcallback.pb.go：

```protobuf
// BindingEventRequest represents input bindings event.
type BindingEventRequest struct {
	// Requried. The name of the input binding component.
	Name string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
	// Required. The payload that the input bindings sent
	Data []byte `protobuf:"bytes,2,opt,name=data,proto3" json:"data,omitempty"`
	// The metadata set by the input binging components.
	Metadata             map[string]string `protobuf:"bytes,3,rep,name=metadata,proto3" json:"metadata,omitempty" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

### BindingEventResponse的定义

pkg/proto/runtime/v1/appcallback.pb.go：

```protobuf
// BindingEventResponse includes operations to save state or
// send data to output bindings optionally.
type BindingEventResponse struct {
	// The name of state store where states are saved.
	StoreName string `protobuf:"bytes,1,opt,name=store_name,json=storeName,proto3" json:"store_name,omitempty"`
	// The state key values which will be stored in store_name.
	States []*v1.StateItem `protobuf:"bytes,2,rep,name=states,proto3" json:"states,omitempty"`
	// The list of output bindings.
	To []string `protobuf:"bytes,3,rep,name=to,proto3" json:"to,omitempty"`
	// The content which will be sent to "to" output bindings.
	Data []byte `protobuf:"bytes,4,opt,name=data,proto3" json:"data,omitempty"`
	// The concurrency of output bindings to send data to
	// "to" output bindings list. The default is SEQUENTIAL.
	Concurrency          BindingEventResponse_BindingEventConcurrency `protobuf:"varint,5,opt,name=concurrency,proto3,enum=dapr.proto.runtime.v1.BindingEventResponse_BindingEventConcurrency" json:"concurrency,omitempty"`
	XXX_NoUnkeyedLiteral struct{}                                     `json:"-"`
	XXX_unrecognized     []byte                                       `json:"-"`
	XXX_sizecache        int32                                        `json:"-"`
}
```



> 备注：只是在proto定义的字段上增加了一些 XXX_ 字段。