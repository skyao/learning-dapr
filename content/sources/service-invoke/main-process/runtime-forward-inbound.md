---
title: "Dapr Runtime转发inbound请求"
linkTitle: "Runtime转发inbound请求"
weight: 80
date: 2021-01-31
description: >
  服务器端的Dapr Runtime将inbound请求转发给服务器端的应用
---

## 协议和端口的配置

Dapr runtime 将 inbound 请求转发给服务器端应用:

```plantuml
title Daprd-Daprd Communication
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
participant user_code_server [
    =App-2
    ----
    server
]

daprd_client -[#red]> daprd_server : Dapr gRPC internal API (remote call)
daprd_server -[#blue]> user_code_server : Dapr HTTP channel API (localhost)
note right: HTTP endpoint @ 3000\nVERB http://localhost:3000/method?query1=value1
daprd_server -[#blue]> user_code_server : Dapr gRPC channel API (localhost)
note right: gRPC endpoint @ 3000\n/dapr.proto.runtime.v1.AppCallback/OnInvoke
```

- app channel 的通讯协议可以是 HTTP 或者 gRPC 协议，可以通过命令行参数 `app-port` 指定，默认是 HTTP
- 应用接收请求的端口可以通过命令行参数 `app-protocol` 指定，没有默认值。
- 为了控制对应用造成的压力，还引入了最大并发度的概念，可以通过命令行参数 `app-max-concurrency` 指定。

## 请求发送的流程

前面分析过，当 internal invoke 的 gRPC 请求进来后，就会进入 `pkc/grpc/api.go` 中的 CallLocal 方法：

```go
func (a *api) CallLocal(ctx context.Context, in *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error) {
	......
	resp, err := a.appChannel.InvokeMethod(ctx, req)
  ......
}
```

然后通过 appChannel 发送请求。

### app channel 的建立

app channel 的建立是在 runtime 初始化时，在 `pkg/runtime/runtime.go` 的 initRuntime() 方法中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    ......
    a.blockUntilAppIsReady()

	err = a.createAppChannel()
	a.daprHTTPAPI.SetAppChannel(a.appChannel)
	grpcAPI.SetAppChannel(a.appChannel)
    ......
}
```

createAppChannel() 的实现，目前只支持 HTTP 和 gRPC：

```go
func (a *DaprRuntime) createAppChannel() error {
    // 为了建立 app channel，必须配置有 app port
	if a.runtimeConfig.ApplicationPort > 0 {
		var channelCreatorFn func(port, maxConcurrency int, spec config.TracingSpec, sslEnabled bool, maxRequestBodySize int, readBufferSize int) (channel.AppChannel, error)

		switch a.runtimeConfig.ApplicationProtocol {
		case GRPCProtocol:
			channelCreatorFn = a.grpc.CreateLocalChannel
		case HTTPProtocol:
			channelCreatorFn = http_channel.CreateLocalChannel
		default:
      // 只支持 HTTP 和 gRPC
			return errors.Errorf("cannot create app channel for protocol %s", string(a.runtimeConfig.ApplicationProtocol))
		}

		ch, err := channelCreatorFn(a.runtimeConfig.ApplicationPort, a.runtimeConfig.MaxConcurrency, a.globalConfig.Spec.TracingSpec, a.runtimeConfig.AppSSL, a.runtimeConfig.MaxRequestBodySize, a.runtimeConfig.ReadBufferSize)
		a.appChannel = ch
	} else {
		log.Warn("app channel is not initialized. did you make sure to configure an app-port?")
	}

	return nil
}
```

#### app channel 的配置参数

和 app channel 密切相关的三个配置项，可以从命令行参数中获取：

```go
func FromFlags() (*DaprRuntime, error) {
    ......
    appPort := flag.String("app-port", "", "The port the application is listening on")
	appProtocol := flag.String("app-protocol", string(HTTPProtocol), "Protocol for the application: grpc or http")	
	appMaxConcurrency := flag.Int("app-max-concurrency", -1, "Controls the concurrency level when forwarding requests to user code")
```

TracingSpec / AppSSL / MaxRequestBodySize / ReadBufferSize 后面细说，先不展开。

### HTTP 通道的实现

HTTP Channel 的实现在文件 `pkg/channel/http/http_channel.go` 中，其 InvokeMethod()方法：

```go
func (h *Channel) InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  ......
	switch req.APIVersion() {
	case internalv1pb.APIVersion_V1:
		rsp, err = h.invokeMethodV1(ctx, req)
  ......
	return rsp, err
}
```

暂时只有 invokeMethodV1 版本：

```go
func (h *Channel) invokeMethodV1(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // 1. 构建HTTP请求
	channelReq := h.constructRequest(ctx, req)
  // 2. 发送请求到应用
	err := h.client.DoTimeout(channelReq, resp, channel.DefaultChannelRequestTimeout)
  // 3. 处理返回的应答
	rsp := h.parseChannelResponse(req, resp, err)
	return rsp, nil
}
```

这是将收到的请求内容，转成HTTP协议的标准格式，然后通过 fasthttp 发给用户代码。其中转为标准http请求的代码在方法 constructRequest() 中：

```go
func (h *Channel) constructRequest(ctx context.Context, req *invokev1.InvokeMethodRequest) *fasthttp.Request {
	var channelReq = fasthttp.AcquireRequest()

	// Construct app channel URI: VERB http://localhost:3000/method?query1=value1
	uri := fmt.Sprintf("%s/%s", h.baseAddress, req.Message().GetMethod())
	channelReq.SetRequestURI(uri)
	channelReq.URI().SetQueryString(req.EncodeHTTPQueryString())
	channelReq.Header.SetMethod(req.Message().HttpExtension.Verb.String())

	// Recover headers
	invokev1.InternalMetadataToHTTPHeader(ctx, req.Metadata(), channelReq.Header.Set)

  ......
}
```

这样在服务器端的用户代码中，就可以用不引入 dapr sdk，只需要提供标准 http endpoint 即可。

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

### gRPC 通道的实现

`pkg/grpc/grpc.go` 中的 CreateLocalChannel() 方法：

```go
// CreateLocalChannel creates a new gRPC AppChannel.
func (g *Manager) CreateLocalChannel(port, maxConcurrency int, spec config.TracingSpec, sslEnabled bool, maxRequestBodySize int, readBufferSize int) (channel.AppChannel, error) {
  // IP地址写死了 127.0.0.1
	conn, err := g.GetGRPCConnection(context.TODO(), fmt.Sprintf("127.0.0.1:%v", port), "", "", true, false, sslEnabled)
  ......
	g.AppClient = conn
	ch := grpc_channel.CreateLocalChannel(port, maxConcurrency, conn, spec, maxRequestBodySize, readBufferSize)
	return ch, nil
}
```

实现代码在 `pkg/channel/grpc/grpc_channel.go` 的 InvokeMethod()方法中：

```go
func (g *Channel) InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  ......
	switch req.APIVersion() {
	case internalv1pb.APIVersion_V1:
		rsp, err = g.invokeMethodV1(ctx, req)
  ......
	return rsp, err
}
```

暂时只有 invokeMethodV1 版本：

```go
func (g *Channel) invokeMethodV1(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // 1. 创建 AppCallback 的 grpc client
	clientV1 := runtimev1pb.NewAppCallbackClient(g.client)
  // 2. 调用 AppCallback 的 OnInvoke() 方法
	resp, err := clientV1.OnInvoke(ctx, req.Message(), grpc.Header(&header), grpc.Trailer(&trailer))
  // 3. 处理返回的应答
	return rsp.WithMessage(resp), nil
}
```

gRPC channel 是通过 gRPC 协议调用服务器端应用上的 gRPC 服务完成，具体是 AppCallback 的 OnInvoke() 方法。

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

也就是说：如果要支持 gRPC channel，则要求服务器端应用必须实现 AppCallback gRPC 服务器，这一点和 HTTP 不同，对服务器端应用是有侵入的。
