---
title: "服务器端App接收inbound请求"
linkTitle: "App接收inbound请求"
weight: 90
date: 2021-04-01
description: >
  服务器端App接收标准HTTP请求，或者实现AppCallbackServer以接受gRPC请求
---

`pkg/proto/runtime/v1/appcallback.pb.go` 中的 OnInvoke 方法：

```go
// AppCallbackServer is the server API for AppCallback service.
type AppCallbackServer interface {
	// Invokes service method with InvokeRequest.
	OnInvoke(context.Context, *v1.InvokeRequest) (*v1.InvokeResponse, error)
}
```

为了接收来自daprd转发的来自客户端的service invoke 请求，服务器端的应用也需要做一些处理。



## 接收HTTP请求

对于通过 HTTP channel 过来的标准HTTP请求，服务器端的应用只需要提供标准的HTTP端口即可，无须引入dapr SDK。

```plantuml
title Daprd-Daprd Communication
hide footbox
skinparam style strictuml

participant daprd_server [
    =daprd
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]

daprd_server -[#blue]> user_code_server : HTTP (localhost)
note right: HTTP endpoint @ 3000\nVERB http://localhost:3000/method?query1=value1
```

## 接收gRPC请求

对于通过 gRPC channel 过来的 gRPC 请求，服务器端的应用则需要实现 gRPC AppCallback 服务的 OnInvoke() 方法：

```plantuml
title Dapr gRPC Channel
hide footbox
skinparam style strictuml

participant daprd_server [
    =daprd
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]


daprd_server -[#blue]> user_code_server : gRPC (localhost)
note right: gRPC endpoint @ 3000\n/dapr.proto.runtime.v1.AppCallback/OnInvoke
```

AppCallbackServer 的 proto 定义在 dapr 仓库下的文件`dapr/proto/runtime/v1/appcallback.proto`中：

```protobuf
service AppCallback {
  // Invokes service method with InvokeRequest.
  rpc OnInvoke (common.v1.InvokeRequest) returns (common.v1.InvokeResponse) {}
  ......
}
```

而 AppCallbackServer 的具体实现则分布在各个不同语言的 sdk 里面。

### go-sdk实现

实现在 go-sdk 的 `service/grpc/invoke.go` 文件的 OnInvoke方法，主要流程为：

```go
func (s *Server) OnInvoke(ctx context.Context, in *cpb.InvokeRequest) (*cpb.InvokeResponse, error) {
	if fn, ok := s.invokeHandlers[in.Method]; ok {
		e := &cc.InvocationEvent{}
		ct, er := fn(ctx, e)
		return &cpb.InvokeResponse{......}, nil
	}
	return nil, fmt.Errorf("method not implemented: %s", in.Method)
}
```

其中 `s.invokeHandlers` 中保存处理请求的方法（由参数method作为key）。AddServiceInvocationHandler() 用于增加方法名和 handler 的映射 ：

```go
// Server is the gRPC service implementation for Dapr.
type Server struct {
	invokeHandlers  map[string]common.ServiceInvocationHandler
}
type  ServiceInvocationHandler func(ctx context.Context, in *InvocationEvent) (out *Content, err error)

func (s *Server) AddServiceInvocationHandler(method string, fn func(ctx context.Context, in *cc.InvocationEvent) (our *cc.Content, err error)) error {
	s.invokeHandlers[method] = fn
	return nil
}
```

这意味着，在服务器端的应用中，并不需要为这些方法提供 gRPC 相关的 proto 定义，也不需要直接通过 gRPC 把这些方法暴露出去，只需要实现 AppCallback 的 OnInvode() 方法，然后把需要对外暴露的方法注册即可，OnInvode() 方法相当于一个简单的 API 网管。

```plantuml
title Dapr AppCallback OnInvoke gRPC impl
hide footbox
skinparam style strictuml

participant AppCallback [
    =AppCallback
    ----
    OnInvoke()
]

participant invokeHandlers
participant handler

-[#blue]> AppCallback : gRPC OnInvode()
note right: gRPC endpoint @ 3000\n/dapr.proto.runtime.v1.AppCallback/OnInvoke
AppCallback -> invokeHandlers: find handler by method name
invokeHandlers --> AppCallback: registered handler
AppCallback -> handler: call handler
note right: type  ServiceInvocationHandler \nfunc(ctx context.Context, in *InvocationEvent) \n(out *Content, err error)
handler --> AppCallback
<-[#blue]- AppCallback
```

#### 用户代码实现示例

用户在开发支持 dapr 的 go 服务器端应用时，需要在应用中启动 dapr service server，然后添加各种 handler，包括 ServiceInvocationHandler，如下面这个例子（go-sdk下的 `example/serving/grpc/main.go` ）：

```go
func main() {
	// create a Dapr service server
	s, err := daprd.NewService(":50001")

	// add a service to service invocation handler
	if err := s.AddServiceInvocationHandler("echo", echoHandler); err != nil {
		log.Fatalf("error adding invocation handler: %v", err)
	}

	// start the server
	if err := s.Start(); err != nil {
		log.Fatalf("server error: %v", err)
	}
}
```


### java-sdk实现

java SDK 中没有找到服务器端实现的代码？待确定。



