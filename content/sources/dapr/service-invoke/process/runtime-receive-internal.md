---
type: docs
title: "服务调用的runtime接收请求"
linkTitle: "Runtime 接收内部请求"
weight: 60
date: 2021-01-31
description: >
  Dapr服务调用的客户端sdk封装
---

Dapr runtime 之间相互通讯走的是 gRPC internal API，这个 API 也只支持 gRPC 方法。

## gRPC Internal API

### 概述

Runtime 初始化时，在注册 gRPC 服务时绑定了 gPRC Internal API 实现和 CallLocal gRPC 方法。

当 internal invoke 的 gRPC 请求进来后，就会进入 `pkc/grpc/api.go` 中的 CallLocal 方法：

```go
func (a *api) CallLocal(ctx context.Context, in *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error) {
	// 1. 构造请求
	req, err := invokev1.InternalInvokeRequest(in)
	// 2. 通过 appChannel 向应用发出请求
	resp, err := a.appChannel.InvokeMethod(ctx, req)
    // 3. 处理应答
	return resp.Proto(), err
}
```

处理方式很清晰，基本上就是将请求通过 app channel 转发。

### app channel 的建立

app channel 的建立是在 runtime 初始化时，在 `pkg/runtime/runtime.go` 的 initRuntime() 方法中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    ......
    a.blockUntilAppIsReady()

	err = a.createAppChannel()
	if err != nil {
		log.Warnf("failed to open %s channel to app: %s", string(a.runtimeConfig.ApplicationProtocol), err)
	}
	a.daprHTTPAPI.SetAppChannel(a.appChannel)
	grpcAPI.SetAppChannel(a.appChannel)
    ......
}

```

createAppChannel() 的实现：

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
			return errors.Errorf("cannot create app channel for protocol %s", string(a.runtimeConfig.ApplicationProtocol))
		}

		ch, err := channelCreatorFn(a.runtimeConfig.ApplicationPort, a.runtimeConfig.MaxConcurrency, a.globalConfig.Spec.TracingSpec, a.runtimeConfig.AppSSL, a.runtimeConfig.MaxRequestBodySize, a.runtimeConfig.ReadBufferSize)
		if err != nil {
			log.Infof("app max concurrency set to %v", a.runtimeConfig.MaxConcurrency)
		}
		a.appChannel = ch
	} else {
		log.Warn("app channel is not initialized. did you make sure to configure an app-port?")
	}

	return nil
}
```

### app channel 的配置参数

和 app channel 相关的三个配置项，可以从命令行参数中获取：

```go
func FromFlags() (*DaprRuntime, error) {
    ......
    appPort := flag.String("app-port", "", "The port the application is listening on")
	appProtocol := flag.String("app-protocol", string(HTTPProtocol), "Protocol for the application: grpc or http")	
	appMaxConcurrency := flag.Int("app-max-concurrency", -1, "Controls the concurrency level when forwarding requests to user code")
```

appPort 必须设置， appProtocol 和 appMaxConcurrency 相关细节处理有点费解：

```go
    var concurrency int			// 这里默认值为0,语义不一致，好在后面的处理是判断 > 0
	if *appMaxConcurrency != -1 {
		concurrency = *appMaxConcurrency
	}

	appPrtcl := string(HTTPProtocol)
	if *appProtocol != string(HTTPProtocol) {
		appPrtcl = *appProtocol
	}
```

### gRPC 通道的实现

`pkg/grpc/grpc.go` 中的 CreateLocalChannel() 方法：

```go
// CreateLocalChannel creates a new gRPC AppChannel.
func (g *Manager) CreateLocalChannel(port, maxConcurrency int, spec config.TracingSpec, sslEnabled bool, maxRequestBodySize int, readBufferSize int) (channel.AppChannel, error) {
	conn, err := g.GetGRPCConnection(context.TODO(), fmt.Sprintf("127.0.0.1:%v", port), "", "", true, false, sslEnabled)
	if err != nil {
		return nil, errors.Errorf("error establishing connection to app grpc on port %v: %s", port, err)
	}

	g.AppClient = conn
	ch := grpc_channel.CreateLocalChannel(port, maxConcurrency, conn, spec, maxRequestBodySize, readBufferSize)
	return ch, nil
}
```

`pkg/channel/grpc/grpc_channel.go` 中的 CreateLocalChannel():

```go
// CreateLocalChannel creates a gRPC connection with user code.
func CreateLocalChannel(port, maxConcurrency int, conn *grpc.ClientConn, spec config.TracingSpec, maxRequestBodySize int, readBufferSize int) *Channel {
	c := &Channel{
		client:             conn,
		baseAddress:        fmt.Sprintf("%s:%d", channel.DefaultChannelAddress, port),
		tracingSpec:        spec,
		appMetadataToken:   auth.GetAppToken(),
		maxRequestBodySize: maxRequestBodySize,
		readBufferSize:     readBufferSize,
	}
	if maxConcurrency > 0 {
		c.ch = make(chan int, maxConcurrency)
	}
	return c
}
```

### HTTP 通道的实现

先忽略
