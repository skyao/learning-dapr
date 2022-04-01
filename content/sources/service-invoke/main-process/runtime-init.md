---
type: docs
title: "服务调用相关的Runtime初始化"
linkTitle: "Runtime初始化"
weight: 20
date: 2021-01-31
description: >
  Dapr Runtime中和服务调用相关的初始化流程
---

在 dapr runtime 启动进行初始化时，需要开启 API 端口并挂载相应的 handler 来接收并处理服务调用的 outbound 请求。另外为了接收来自其他 dapr runtime 的 inbound 请求，还要启动 dapr internal server。

## Dapr HTTP API Server(outbound)

### 在 dapr runtime 中启动 HTTP server

dapr runtime 的 HTTP server 用的是 fasthttp。

在 dapr runtime 启动时的初始化过程中，会启动 HTTP server， 代码在 `pkg/runtime/runtime.go` 中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
  ......
  // Start HTTP Server
	err = a.startHTTPServer(a.runtimeConfig.HTTPPort, a.runtimeConfig.PublicPort, a.runtimeConfig.ProfilePort, a.runtimeConfig.AllowedOrigins, pipeline)
	if err != nil {
		log.Fatalf("failed to start HTTP server: %s", err)
	}
  ......
}

func (a *DaprRuntime) startHTTPServer(......) error {
	a.daprHTTPAPI = http.NewAPI(......)

	server := http.NewServer(a.daprHTTPAPI, ......)
  if err := server.StartNonBlocking(); err != nil {		// StartNonBlocking 启动 fasthttp server
		return err
	}
}
```

StartNonBlocking() 的实现代码在 `pkg/http/server.go` 中：

```go
// StartNonBlocking starts a new server in a goroutine.
func (s *server) StartNonBlocking() error {
  	......
  	for _, apiListenAddress := range s.config.APIListenAddresses {
			l, err := net.Listen("tcp", fmt.Sprintf("%s:%v", apiListenAddress, s.config.Port))
      listeners = append(listeners, l)
		}
  
  	for _, listener := range listeners {
		// customServer is created in a loop because each instance
		// has a handle on the underlying listener.
		customServer := &fasthttp.Server{
			Handler:            handler,
			MaxRequestBodySize: s.config.MaxRequestBodySize * 1024 * 1024,
			ReadBufferSize:     s.config.ReadBufferSize * 1024,
			StreamRequestBody:  s.config.StreamRequestBody,
		}
		s.servers = append(s.servers, customServer)

		go func(l net.Listener) {
			if err := customServer.Serve(l); err != nil {
				log.Fatal(err)
			}
		}(listener)
	}
}
```

### 挂载 DirectMessaging 的 HTTP  端点

在 HTTP API 的初始化过程中，会在 fast http server 上挂载 DirectMessaging 的 HTTP  端点，代码在 `pkg/http/api.go` 中：

```go
func NewAPI(
  appID string,
	appChannel channel.AppChannel,
	directMessaging messaging.DirectMessaging,
  ......
  	shutdown func()) API {
  
  	api := &api{
		appChannel:               appChannel,
		directMessaging:          directMessaging,
		......
	}
  
  	// 附加 DirectMessaging 的 HTTP 端点
  	api.endpoints = append(api.endpoints, api.constructDirectMessagingEndpoints()...)
}
```

DirectMessaging 的 HTTP 端点的具体信息在 constructDirectMessagingEndpoints() 方法中：

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

注意这里的 Route 路径 "invoke/{id}/method/{method:*}"， dapr sdk 就是就通过这样的 url 来发起 HTTP 请求。



```plantuml
title Dapr HTTP API 
hide footbox
skinparam style strictuml

participant daprd_client [
    =daprd
    ----
    client
]

-[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500\n/v1.0/invoke/{id}/method/{method}
|||
<[#blue]-- daprd_client
```

## Dapr gRPC API Server(outbound)

### 启动 gRPC 服务器

在 dapr runtime 启动时的初始化过程中，会启动 gRPC server， 代码在 `pkg/runtime/runtime.go` 中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    // Create and start internal and external gRPC servers
	grpcAPI := a.getGRPCAPI()
    
	err = a.startGRPCAPIServer(grpcAPI, a.runtimeConfig.APIGRPCPort)
    ......
}

func (a *DaprRuntime) startGRPCAPIServer(api grpc.API, port int) error {
	serverConf := a.getNewServerConfig(a.runtimeConfig.APIListenAddresses, port)
	server := grpc.NewAPIServer(api, serverConf, a.globalConfig.Spec.TracingSpec, a.globalConfig.Spec.MetricSpec, a.globalConfig.Spec.APISpec, a.proxy)
    if err := server.StartNonBlocking(); err != nil {
		return err
	}
	......
}

// NewAPIServer returns a new user facing gRPC API server.
func NewAPIServer(api API, config ServerConfig, ......) Server {
	return &server{
		api:         api,
		config:      config,
		kind:        apiServer, // const apiServer = "apiServer"
		......
	}
}
```

### 注册 Dapr API

为了让 dapr runtime 的 gRPC  服务器能挂载 Dapr API，需要进行注册上去。

注册的代码实现在 `pkg/grpc/server.go` 中， StartNonBlocking() 方法在启动 grpc 服务器时，会进行服务注册：


```go
func (s *server) StartNonBlocking() error {
		if s.kind == internalServer {
			internalv1pb.RegisterServiceInvocationServer(server, s.api)
		} else if s.kind == apiServer {
            runtimev1pb.RegisterDaprServer(server, s.api)		// 注意：s.api (即 gRPC api 实现) 被传递进去
		}
		......
}
```

而 RegisterDaprServer() 方法的实现代码在 `pkg/proto/runtime/v1/dapr_grpc.pb.go`:

```go
func RegisterDaprServer(s grpc.ServiceRegistrar, srv DaprServer) {
	s.RegisterService(&Dapr_ServiceDesc, srv)					// srv 即 gRPC api 实现
}
```

### Dapr_ServiceDesc 定义

在文件 `pkg/proto/runtime/v1/dapr_grpc.pb.go` 中有 Dapr Service 的 grpc 服务定义，这是 protoc 生成的 gRPC 代码。

Dapr_ServiceDesc 中有 Dapr Service 各个方法的定义，和服务调用相关的是 `InvokeService` 方法：

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

```plantuml
title Dapr gRPC API 
hide footbox
skinparam style strictuml

participant daprd_client [
    =daprd
    ----
    client
]

-[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001\n/dapr.proto.runtime.v1.Dapr/InvokeService
|||
<[#blue]-- daprd_client
```

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
		return srv.(DaprServer).InvokeService(ctx, req.(*InvokeServiceRequest))		// 这里调用的 srv 即 gRPC api 实现
	}
	return interceptor(ctx, in, info, handler)
}
```

最后调用到了 DaprServer 接口实现的 InvokeService 方法，也就是 gPRC API 实现。

## Dapr Internal API Server(inbound)

### 启动 gRPC 服务器

在 dapr runtime 启动时的初始化过程中，会启动 gRPC internal server， 代码在 `pkg/runtime/runtime.go` 中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
	err = a.startGRPCInternalServer(grpcAPI, a.runtimeConfig.InternalGRPCPort)
	if err != nil {
		log.Fatalf("failed to start internal gRPC server: %s", err)
	}
	log.Infof("internal gRPC server is running on port %v", a.runtimeConfig.InternalGRPCPort)
    ......
}

func (a *DaprRuntime) startGRPCInternalServer(api grpc.API, port int) error {
	serverConf := a.getNewServerConfig([]string{""}, port)
	server := grpc.NewInternalServer(api, serverConf, a.globalConfig.Spec.TracingSpec, a.globalConfig.Spec.MetricSpec, a.authenticator, a.proxy)
	if err := server.StartNonBlocking(); err != nil {
		return err
	}
	a.apiClosers = append(a.apiClosers, server)

	return nil
}
```

### 特殊处理：端口

grpc internal server 的端口比较特殊，可以通过命令行参数 "--dapr-internal-grpc-port" 指定，而如果没有指定，是取一个随机的可用端口，而不是取某个固定值。这一点和 dapr HTTP api server 以及 dapr gRPC api server 不同。

具体代码实现在文件 `pkg/runtime/cli.go` 中：

```go
func FromFlags() (*DaprRuntime, error) {	
	var daprInternalGRPC int
	if *daprInternalGRPCPort != "" {
		daprInternalGRPC, err = strconv.Atoi(*daprInternalGRPCPort)
		if err != nil {
			return nil, errors.Wrap(err, "error parsing dapr-internal-grpc-port")
		}
	} else {
		daprInternalGRPC, err = grpc.GetFreePort()
		if err != nil {
			return nil, errors.Wrap(err, "failed to get free port for internal grpc server")
		}
	}
    ......
}
```

### 特殊处理：重用 gRPC API handler

Dapr gRPC internal API 实现时有点特殊： 

- 启动了自己的 gRPC  server，也有自己的端口。
- 但是注册的负责处理请求的 handler 却重用了 Dapr gRPC internal API 

darp runtime 的初始化代码中，grpcAPI 对象是 GRPC API Server 和 GRPC Internal Server 共用的：

```go
grpcAPI := a.getGRPCAPI()

err = a.startGRPCAPIServer(grpcAPI, a.runtimeConfig.APIGRPCPort)
err = a.startGRPCInternalServer(grpcAPI, a.runtimeConfig.InternalGRPCPort)
```

从设计的角度看，这样做不好：混淆了对 outbound 请求和 inbound 请求的处理，影响代码可读性。

### 注册 Dapr API

为了让 dapr runtime 的 gRPC  服务器能挂载 Dapr internal API，需要进行注册。

注册的代码实现在 `pkg/grpc/server.go` 中， StartNonBlocking() 方法在启动 grpc 服务器时，会进行服务注册：


```go
func (s *server) StartNonBlocking() error {
		if s.kind == internalServer {
			internalv1pb.RegisterServiceInvocationServer(server, s.api)		// 注意：s.api (即 gRPC api 实现) 被传递进去
		} else if s.kind == apiServer {
			runtimev1pb.RegisterDaprServer(server, s.api)
		}
		......
}
```

而 RegisterServiceInvocationServer() 方法的实现代码在 `pkg/proto/internals/v1/service_invocation_grpc.pb.go`:

```go
func RegisterServiceInvocationServer(s grpc.ServiceRegistrar, srv ServiceInvocationServer) {
	s.RegisterService(&ServiceInvocation_ServiceDesc, srv)  					// srv 即 gRPC api 实现
}
```

### ServiceInvocation_ServiceDesc 定义

在文件 `pkg/proto/internals/v1/service_invocation_grpc.pb.go` 中有 internal Service 的 grpc 服务定义，这是 protoc 生成的 gRPC 代码。

ServiceInvocation_ServiceDesc 中有两个方法的定义，和服务调用相关的是 `CallLocal` 方法：

```go
var ServiceInvocation_ServiceDesc = grpc.ServiceDesc{
	ServiceName: "dapr.proto.internals.v1.ServiceInvocation",
	HandlerType: (*ServiceInvocationServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "CallActor",
			Handler:    _ServiceInvocation_CallActor_Handler,
		},
		{
			MethodName: "CallLocal",
			Handler:    _ServiceInvocation_CallLocal_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "dapr/proto/internals/v1/service_invocation.proto",
}
```

这一段是告诉 gRPC server： 如果收到访问 `dapr.proto.internals.v1.ServiceInvocation` 服务的 `CallLocal` 方法的  gRPC 请求，请把请求转给 `_ServiceInvocation_CallLocal_Handler` 处理。

```plantuml
title Dapr gRPC internal API 
hide footbox
skinparam style strictuml

participant daprd_client [
    =daprd
    ----
    client
]

-[#red]> daprd_client : gRPC (remote call)
note right: gRPC API @ ramdon port\n/dapr.proto.internals.v1.ServiceInvocation/CallLocal
|||
<[#red]-- daprd_client
```

而 `CallLocal` 方法相关联的 handler 方法 `_ServiceInvocation_CallLocal_Handler `的实现代码是：

```go
func _ServiceInvocation_CallLocal_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(InternalInvokeRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
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

最后调用到了 ServiceInvocationServer 接口实现的 CallLocal 方法，也就是 gPRC  API 实现。

