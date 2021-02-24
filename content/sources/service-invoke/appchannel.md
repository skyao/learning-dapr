---
type: docs
title: "服务调用的runtime转发请求给服务器端"
linkTitle: "转发请求给服务器端"
weight: 545
date: 2021-01-31
description: >
  Dapr服务调用的runtime转发请求给服务器端
---


`pkc/grpc/api.go` 中的 CallLocal 方法：

```go
func (a *api) CallLocal(ctx context.Context, in *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error) {
	if a.appChannel == nil {
		return nil, status.Error(codes.Internal, "app channel is not initialized")
	}

	req, err := invokev1.InternalInvokeRequest(in)
	if err != nil {
		return nil, status.Errorf(codes.InvalidArgument, "parsing InternalInvokeRequest error: %s", err.Error())
	}

	resp, err := a.appChannel.InvokeMethod(ctx, req)

	if err != nil {
		return nil, err
	}
	return resp.Proto(), err
}
```

## App Channel的定义

app channel 定义在 `pkg/channel/channel.go` 中，AppChannel 是和用户代码通讯的抽象层：

```go
type AppChannel interface {
	GetBaseAddress() string
	InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error)
}
```

## Channel实现

channel有两个实现： grpc channel 和 http channel。

### grpc channel

`pkg/channel/grpc/grpc_channel.go` 的 InvokeMethod()方法：

```go
func (g *Channel) InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	var rsp *invokev1.InvokeMethodResponse
	var err error

	switch req.APIVersion() {
	case internalv1pb.APIVersion_V1:
		rsp, err = g.invokeMethodV1(ctx, req)

	default:
		// Reject unsupported version
		rsp = nil
		err = status.Error(codes.Unimplemented, fmt.Sprintf("Unsupported spec version: %d", req.APIVersion()))
	}

	return rsp, err
}
```

暂时只有 invokeMethodV1 版本：

```go
func (g *Channel) invokeMethodV1(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	if g.ch != nil {
		g.ch <- 1
	}

	clientV1 := runtimev1pb.NewAppCallbackClient(g.client)
	grpcMetadata := invokev1.InternalMetadataToGrpcMetadata(ctx, req.Metadata(), true)
	// Prepare gRPC Metadata
	ctx = metadata.NewOutgoingContext(context.Background(), grpcMetadata)

	ctx, cancel := context.WithTimeout(ctx, channel.DefaultChannelRequestTimeout)
	defer cancel()
	var header, trailer metadata.MD
	resp, err := clientV1.OnInvoke(ctx, req.Message(), grpc.Header(&header), grpc.Trailer(&trailer))

	if g.ch != nil {
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

### HTTP Channel

HTTP Channel 的实现在文件 `pkg/channel/http/http_channel.go` 中，其 InvokeMethod()方法：

```go
func (h *Channel) InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	// Check if HTTP Extension is given. Otherwise, it will return error.
	httpExt := req.Message().GetHttpExtension()
	if httpExt == nil {
		return nil, status.Error(codes.InvalidArgument, "missing HTTP extension field")
	}
	if httpExt.GetVerb() == commonv1pb.HTTPExtension_NONE {
		return nil, status.Error(codes.InvalidArgument, "invalid HTTP verb")
	}

	var rsp *invokev1.InvokeMethodResponse
	var err error
	switch req.APIVersion() {
	case internalv1pb.APIVersion_V1:
		rsp, err = h.invokeMethodV1(ctx, req)

	default:
		// Reject unsupported version
		err = status.Error(codes.Unimplemented, fmt.Sprintf("Unsupported spec version: %d", req.APIVersion()))
	}

	return rsp, err
}
```

暂时只有 invokeMethodV1 版本：

```go
func (h *Channel) invokeMethodV1(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	channelReq := h.constructRequest(ctx, req)

	if h.ch != nil {
		h.ch <- 1
	}

	// Emit metric when request is sent
	verb := string(channelReq.Header.Method())
	diag.DefaultHTTPMonitoring.ClientRequestStarted(ctx, verb, req.Message().Method, int64(len(req.Message().Data.GetValue())))
	startRequest := time.Now()

	// Send request to user application
	var resp = fasthttp.AcquireResponse()
	err := h.client.DoTimeout(channelReq, resp, channel.DefaultChannelRequestTimeout)
	defer func() {
		fasthttp.ReleaseRequest(channelReq)
		fasthttp.ReleaseResponse(resp)
	}()

	elapsedMs := float64(time.Since(startRequest) / time.Millisecond)

	if h.ch != nil {
		<-h.ch
	}

	rsp := h.parseChannelResponse(req, resp, err)
	diag.DefaultHTTPMonitoring.ClientRequestCompleted(ctx, verb, req.Message().GetMethod(), strconv.Itoa(int(rsp.Status().Code)), int64(resp.Header.ContentLength()), elapsedMs)

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

	// Set Content body and types
	contentType, body := req.RawData()
	channelReq.Header.SetContentType(contentType)
	channelReq.SetBody(body)

	return channelReq
}
```

这样就直接绕开用户代码中的 dapr sdk，直接调用到用户代码中的标准 http endpoint。 



