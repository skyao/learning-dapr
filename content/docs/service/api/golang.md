---
title: "服务调用API的golang生成代码"
linkTitle: "golang生成代码"
weight: 523
date: 2021-01-29
description: >
  服务调用API的golang生成代码
---

从proto api定义文件生成的golang代码，被存放在dapr项目的 ` pkg/proto/` 目录下。

### grpc服务定义

DaprServer 是 dapr 服务的服务器端API定义，包含 InvokeService方法：

```go
// DaprServer is the server API for Dapr service.
type DaprServer interface {
	// Invokes a method on a remote Dapr app.
	InvokeService(context.Context, *InvokeServiceRequest) (*v1.InvokeResponse, error)
   ......
}
```

AppCallbackServer 是 AppCallback 服务的服务器端API定义，包含 OnInvoke 方法：

```go
// AppCallbackServer is the server API for AppCallback service.
type AppCallbackServer interface {
	// Invokes service method with InvokeRequest.
	OnInvoke(context.Context, *v1.InvokeRequest) (*v1.InvokeResponse, error)
	......
}
```

### InvokeServiceRequest的定义

https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/pkg/proto/runtime/v1/dapr.pb.go

```go
type InvokeServiceRequest struct {
	// Required. Callee's app id.
	Id string `protobuf:"bytes,1,opt,name=id,proto3" json:"id,omitempty"`
	// Required. message which will be delivered to callee.
	Message              *v1.InvokeRequest `protobuf:"bytes,3,opt,name=message,proto3" json:"message,omitempty"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

### InvokeRequest的定义

https://github.com/dapr/dapr/blob/de49fe260c8f7c53e146e27150faad8c0880fe90/pkg/proto/common/v1/common.pb.go

```go
type InvokeRequest struct {
	// Required. method is a method name which will be invoked by caller.
	Method string `protobuf:"bytes,1,opt,name=method,proto3" json:"method,omitempty"`
	// Required. Bytes value or Protobuf message which caller sent.
	// Dapr treats Any.value as bytes type if Any.type_url is unset.
	Data *any.Any `protobuf:"bytes,2,opt,name=data,proto3" json:"data,omitempty"`
	// The type of data content.
	//
	// This field is required if data delivers http request body
	// Otherwise, this is optional.
	ContentType string `protobuf:"bytes,3,opt,name=content_type,json=contentType,proto3" json:"content_type,omitempty"`
	// HTTP specific fields if request conveys http-compatible request.
	//
	// This field is required for http-compatible request. Otherwise,
	// this field is optional.
	HttpExtension        *HTTPExtension `protobuf:"bytes,4,opt,name=http_extension,json=httpExtension,proto3" json:"http_extension,omitempty"`
	XXX_NoUnkeyedLiteral struct{}       `json:"-"`
	XXX_unrecognized     []byte         `json:"-"`
	XXX_sizecache        int32          `json:"-"`
}
```

### InvokeResponse的定义

```go
type InvokeResponse struct {
	// Required. The content body of InvokeService response.
	Data *any.Any `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
	// Required. The type of data content.
	ContentType          string   `protobuf:"bytes,2,opt,name=content_type,json=contentType,proto3" json:"content_type,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```



> 备注：只是在proto定义的字段上增加了一些 XXX_ 字段。