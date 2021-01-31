---
title: "服务调用的runtime转发请求"
linkTitle: "Runtime转发请求"
weight: 544
date: 2021-01-31
description: >
  Dapr服务调用的客户端sdk封装
---


`pkc/grpc/api.go` 中的 InvokeService 方法：

```go
func (a *api) InvokeService(ctx context.Context, in *runtimev1pb.InvokeServiceRequest) (*commonv1pb.InvokeResponse, error) {
	req := invokev1.FromInvokeRequestMessage(in.GetMessage())

	if incomingMD, ok := metadata.FromIncomingContext(ctx); ok {
		req.WithMetadata(incomingMD)
	}

	resp, err := a.directMessaging.Invoke(ctx, in.Id, req)
	if err != nil {
		return nil, err
	}

	var headerMD = invokev1.InternalMetadataToGrpcMetadata(ctx, resp.Headers(), true)

	var respError error
	if resp.IsHTTPResponse() {
		var errorMessage = []byte("")
		if resp != nil {
			_, errorMessage = resp.RawData()
		}
		respError = invokev1.ErrorFromHTTPResponseCode(int(resp.Status().Code), string(errorMessage))
		// Populate http status code to header
		headerMD.Set(daprHTTPStatusHeader, strconv.Itoa(int(resp.Status().Code)))
	} else {
		respError = invokev1.ErrorFromInternalStatus(resp.Status())
		// ignore trailer if appchannel uses HTTP
		grpc.SetTrailer(ctx, invokev1.InternalMetadataToGrpcMetadata(ctx, resp.Trailers(), false))
	}

	grpc.SetHeader(ctx, headerMD)

	return resp.Message(), respError
}
```

`pkg/messaging/direct_messaging.go` 的 Invoke()方法：

```go
func (d *directMessaging) Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	if targetAppID == d.appID {
		return d.invokeLocal(ctx, req)
	}
	return d.invokeWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, targetAppID, d.invokeRemote, req)
}

func (d *directMessaging) invokeRemote(ctx context.Context, targetID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	address, err := d.getAddressFromMessageRequest(targetID)
	if err != nil {
		return nil, err
	}

	conn, err := d.connectionCreatorFn(address, targetID, false, false)
	if err != nil {
		return nil, err
	}

	span := diag_utils.SpanFromContext(ctx)

	// TODO: Use built-in grpc client timeout instead of using context timeout
	ctx, cancel := context.WithTimeout(ctx, channel.DefaultChannelRequestTimeout)
	defer cancel()

	// no ops if span context is empty
	ctx = diag.SpanContextToGRPCMetadata(ctx, span.SpanContext())

	d.addForwardedHeadersToMetadata(req)
	d.addDestinationAppIDHeaderToMetadata(targetID, req)

	clientV1 := internalv1pb.NewServiceInvocationClient(conn)
	resp, err := clientV1.CallLocal(ctx, req.Proto())
	if err != nil {
		return nil, err
	}

	return invokev1.InternalInvokeResponse(resp)
}
```







