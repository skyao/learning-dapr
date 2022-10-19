---
title: "使用方式"
linkTitle: "使用方式"
weight: 2
date: 2021-03-17
description: >
  命名解析在service invoke 流程中的使用方式
---

### 解析地址

name resolver 被调用的地方只有一个：

```go
func (d *directMessaging) getRemoteApp(appID string) (remoteApp, error) {
  // 从appID中获取id和namespace
  // appID 可能是类似 "appID.namespace" 的格式
	id, namespace, err := d.requestAppIDAndNamespace(appID)
	if err != nil {
		return remoteApp{}, err
	}

  // 执行 resolver 的解析
	request := nr.ResolveRequest{ID: id, Namespace: namespace, Port: d.grpcPort}
	address, err := d.resolver.ResolveID(request)
	if err != nil {
		return remoteApp{}, err
	}

  // 返回 remoteApp 的地址
	return remoteApp{
		namespace: namespace,
		id:        id,
		address:   address,
	}, nil
}
```

解析出来的地址在 directMessaging 的 Invoke() 中使用，用来执行远程调用：

```go
// Invoke takes a message requests and invokes an app, either local or remote.
func (d *directMessaging) Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	app, err := d.getRemoteApp(targetAppID)
	if err != nil {
		return nil, err
	}

  // 如果目标应用的 id 和 namespace 都和 directMessaging 的一致，则执行 invokeLocal()
	if app.id == d.appID && app.namespace == d.namespace {
		return d.invokeLocal(ctx, req)
	}
  
  // 这是在带有重试机制的情况下调用 invokeRemote
	return d.invokeWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, app, d.invokeRemote, req)
}
```

invokeWithRetry() 中忽略重试的代码：

```go
func (d *directMessaging) invokeWithRetry(
	ctx context.Context,
	numRetries int,
	backoffInterval time.Duration,
	app remoteApp,
	fn func(ctx context.Context, appID, namespace, appAddress string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error),
	req *invokev1.InvokeMethodRequest,
) (*invokev1.InvokeMethodResponse, error) {
  
}
```





invokeRemote() 

```go
func (d *directMessaging) invokeRemote(ctx context.Context, appID, namespace, appAddress string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
  // 
	conn, teardown, err := d.connectionCreatorFn(context.TODO(), appAddress, appID, namespace, false, false, false)
	defer teardown()
	if err != nil {
		return nil, err
	}

	ctx = d.setContextSpan(ctx)

	d.addForwardedHeadersToMetadata(req)
	d.addDestinationAppIDHeaderToMetadata(appID, req)

	clientV1 := internalv1pb.NewServiceInvocationClient(conn)

	var opts []grpc.CallOption
	opts = append(opts, grpc.MaxCallRecvMsgSize(d.maxRequestBodySize*1024*1024), grpc.MaxCallSendMsgSize(d.maxRequestBodySize*1024*1024))

	resp, err := clientV1.CallLocal(ctx, req.Proto(), opts...)
	if err != nil {
		return nil, err
	}

	return invokev1.InternalInvokeResponse(resp)
}
```

