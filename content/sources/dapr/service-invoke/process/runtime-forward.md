---
title: "服务调用的runtime转发请求"
linkTitle: "Runtime 相互通讯"
weight: 50
date: 2021-01-31
description: >
  Dapr服务调用的客户端sdk封装
---

`pkg/messaging/direct_messaging.go` 中的 DirectMessaging 负责实现转发请求给远程 dapr runtime。

## 接口

DirectMessaging 接口定义，用来调用远程应用：

```go
// DirectMessaging is the API interface for invoking a remote app.
type DirectMessaging interface {
	Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error)
}
```

只有一个 invoke 方法。

## 实现流程

### 流程概况

invoke 方法的实现：

```go
func (d *directMessaging) Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	app, err := d.getRemoteApp(targetAppID)

	if app.id == d.appID && app.namespace == d.namespace {
		return d.invokeLocal(ctx, req)   // 如果调用的 appid 就是自己的 appid，这个场景好奇怪。忽略这里的代码先
	}
	return d.invokeWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, app, d.invokeRemote, req)
}
```

invokeRemote 方法的代码简化如下：

```go
func (d *directMessaging) invokeRemote(ctx context.Context, appID, namespace, appAddress string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
    // 建立连接
	conn, err := d.connectionCreatorFn(context.TODO(), appAddress, appID, namespace, false, false, false)
    // 构建 gRPC stub 作为 client
	clientV1 := internalv1pb.NewServiceInvocationClient(conn)
    // 调用 gRPC 的 CallLocal 方法发出远程调用请求到另外一个 Dapr runtime
	resp, err := clientV1.CallLocal(ctx, req.Proto(), opts...)
    // 处理应答
	return invokev1.InternalInvokeResponse(resp)
}
```

### 发出 gRPC 请求给远程 dapr runtime

CallLocal() 方法的实现在 `service_invocation_grpc.pb.go` 中，这是 protoc 成生的 gRPC 代码：

```go
func (c *serviceInvocationClient) CallLocal(ctx context.Context, in *InternalInvokeRequest, opts ...grpc.CallOption) (*InternalInvokeResponse, error) {
	out := new(InternalInvokeResponse)
	err := c.cc.Invoke(ctx, "/dapr.proto.internals.v1.ServiceInvocation/CallLocal", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

可以看到这个 gRPC 请求调用的是 dapr.proto.internals.v1.ServiceInvocation 服务的 CallLocal 方法。
