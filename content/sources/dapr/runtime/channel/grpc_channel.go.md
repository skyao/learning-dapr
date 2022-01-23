---
type: docs
title: "grpc_channel.go的源码学习"
linkTitle: "grpc_channel.go"
weight: 1820
date: 2021-04-11
description: >
  AppChannel 的 gRPC 实现。
---

Dapr channel package中的 grpc_channel.go 文件的源码学习，AppChannel 的 gRPC 实现。

### Channel 结构体定义

Channel是一个具体的AppChannel实现，用于与基于gRPC的用户代码进行交互。

```go
// Channel is a concrete AppChannel implementation for interacting with gRPC based user code
type Channel struct {
  // grpc 客户端连接
	client           *grpc.ClientConn
  // user code（应用）的地址
	baseAddress      string
  // 限流用的 go chan
	ch               chan int
	tracingSpec      config.TracingSpec
	appMetadataToken string
}
```

### 创建 Channel 结构体

```go
// CreateLocalChannel creates a gRPC connection with user code
func CreateLocalChannel(port, maxConcurrency int, conn *grpc.ClientConn, spec config.TracingSpec) *Channel {
	c := &Channel{
		client:           conn,
    // baseAddress 就是 "ip:port"
		baseAddress:      fmt.Sprintf("%s:%d", channel.DefaultChannelAddress, port),
		tracingSpec:      spec,
		appMetadataToken: auth.GetAppToken(),
	}
	if maxConcurrency > 0 {
    // 如果有并发控制要求，则创建用于并发控制的go channel
		c.ch = make(chan int, maxConcurrency)
	}
	return c
}
```

### GetBaseAddress 方法

```
// GetBaseAddress returns the application base address
func (g *Channel) GetBaseAddress() string {
   return g.baseAddress
}
```

这个方法用来获取app的基础路径，可以拼接其他的字路径，如：

```go
func (a *actorsRuntime) startAppHealthCheck(opts ...health.Option) {
	healthAddress := fmt.Sprintf("%s/healthz", a.appChannel.GetBaseAddress())
	ch := health.StartEndpointHealthCheck(healthAddress, opts...)
	......
}
```

> 备注：只有 actor 这一个地方用到了这个方法

### InvokeMethod 方法

InvokeMethod 方法通过 gRPC 调用 user code：

```go
// InvokeMethod invokes user code via gRPC
func (g *Channel) InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
   var rsp *invokev1.InvokeMethodResponse
   var err error

   switch req.APIVersion() {
   case internalv1pb.APIVersion_V1:
      // 目前只支持 v1 版本
      rsp, err = g.invokeMethodV1(ctx, req)

   default:
      // Reject unsupported version
      // 其他版本会被拒绝
      rsp = nil
      err = status.Error(codes.Unimplemented, fmt.Sprintf("Unsupported spec version: %d", req.APIVersion()))
   }

   return rsp, err
}
```

invokeMethodV1() 的实现

```go
// invokeMethodV1 calls user applications using daprclient v1
func (g *Channel) invokeMethodV1(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
   if g.ch != nil {
      // 往 ch 里面发一个int，等价于当前并发数量 + 1
      g.ch <- 1
   }

   // 创建一个 app callback 的 client
   clientV1 := runtimev1pb.NewAppCallbackClient(g.client)
   // 将内部 metadata 转为 grpc 的 metadata
   grpcMetadata := invokev1.InternalMetadataToGrpcMetadata(ctx, req.Metadata(), true)

   if g.appMetadataToken != "" {
      grpcMetadata.Set(auth.APITokenHeader, g.appMetadataToken)
   }

   // Prepare gRPC Metadata
   ctx = metadata.NewOutgoingContext(context.Background(), grpcMetadata)

   var header, trailer metadata.MD
   // 调用user code
   resp, err := clientV1.OnInvoke(ctx, req.Message(), grpc.Header(&header), grpc.Trailer(&trailer))

   if g.ch != nil {
      // 从 ch 中读取一个int，等价于当前并发数量 - 1
      // 但这个操作并没有额外保护，如果上面的代码发生 panic，岂不是这个计数器就出错了？
      // 考虑把这个操作放在 deffer 中进行会比较安全
      <-g.ch
   }

   var rsp *invokev1.InvokeMethodResponse
   if err != nil {
      // Convert status code
      respStatus := status.Convert(err)
      // Prepare response
      rsp = invokev1.NewInvokeMethodResponse(int32(respStatus.Code()), respStatus.Message(), respStatus.Proto().Details)
   } else {
      rsp = invokev1.NewInvokeMethodResponse(int32(codes.OK), "", nil)
   }

   rsp.WithHeaders(header).WithTrailers(trailer)

   return rsp.WithMessage(resp), nil
}
```

使用这个方法的地方有：

- actor 的 callLocalActor() 和 deactivateActor()
- Grpc api 中的 CallLocal()
- messaging 中 direct_message 的 invokeLocal()
- runtime中
  * getConfigurationHTTP() 
  * isAppSubscribedToBinding()
  * publishMessageHTTP()
  * sendBindingEventToApp()



