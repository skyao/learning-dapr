---
title: "服务调用的HTTP API"
linkTitle: "HTTP API"
weight: 547
date: 2021-01-31
description: >
  Dapr服务调用的HTTP API
---

### HTTP API Server 初始化

Dapr在  runtime 启动时会启动 Dapr 的 HTTP API server (以及 grpc API server 和用于 dapr runtime相互通讯的 grpc internal server）：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
	......
	err = a.startGRPCAPIServer(grpcAPI, a.runtimeConfig.APIGRPCPort)
	
	err = a.startGRPCInternalServer(grpcAPI, a.runtimeConfig.InternalGRPCPort)
	
	a.startHTTPServer(a.runtimeConfig.HTTPPort, a.runtimeConfig.ProfilePort, a.runtimeConfig.AllowedOrigins, pipeline)
	......
}
```

startHTTPServer的代码：

```go
func (a *DaprRuntime) startHTTPServer(port, profilePort int, allowedOrigins string, pipeline http_middleware.Pipeline) {
	a.daprHTTPAPI = http.NewAPI(a.runtimeConfig.ID, a.appChannel, a.directMessaging, a.stateStores, a.secretStores, a.getPublishAdapter(), a.actor, a.sendToOutputBinding, a.globalConfig.Spec.TracingSpec)
	serverConf := http.NewServerConfig(a.runtimeConfig.ID, a.hostAddress, port, profilePort, allowedOrigins, a.runtimeConfig.EnableProfiling)

	server := http.NewServer(a.daprHTTPAPI, serverConf, a.globalConfig.Spec.TracingSpec, pipeline)
	server.StartNonBlocking()
}
```

NewAPI方法：

```go
func NewAPI(appID string, appChannel channel.AppChannel, directMessaging messaging.DirectMessaging, ......) (*bindings.InvokeResponse, error), tracingSpec config.TracingSpec) API {
	api := &api{
		appChannel:            appChannel,
		directMessaging:       directMessaging,
		......
	}

	api.endpoints = append(api.endpoints, api.constructDirectMessagingEndpoints()...)

	return api
}
```

constructDirectMessagingEndpoints方法构建了接收 service invoke 请求的 HTTP Endpoint：

```go
func (a *api) constructDirectMessagingEndpoints() []Endpoint {
	return []Endpoint{
		{
			Methods: []string{fasthttp.MethodGet, fasthttp.MethodPost, fasthttp.MethodDelete, fasthttp.MethodPut},
			Route:   "invoke/{id}/method/{method:*}",
			Version: apiVersionV1,
			Handler: a.onDirectMessage, // 注意这里注册了 handler
		},
	}
}
```

这就和直接用 curl 等工具给 dapr 发请求的方式对应上了：

`POST/GET/PUT/DELETE http://localhost:<daprPort>/v1.0/invoke/<appId>/method/<method-name>`

这样在 dapr runtime 启动之后，dapr 的 http server 也就启动起来了，然后就有 `"invoke/{id}/method/{method:*}"` 这样的 HTTP Endpoint 可以用来处理请求。

### 处理dapr 请求

`pkg/http/api.go` 的 onDirectMessage 方法：

```go
func (a *api) onDirectMessage(reqCtx *fasthttp.RequestCtx) {
	targetID := reqCtx.UserValue(idParam).(string)
	verb := strings.ToUpper(string(reqCtx.Method()))
	invokeMethodName := reqCtx.UserValue(methodParam).(string)
	if invokeMethodName == "" {
		msg := NewErrorResponse("ERR_DIRECT_INVOKE", "invalid method name")
		respondWithError(reqCtx, fasthttp.StatusBadRequest, msg)
		log.Debug(msg)
		return
	}

	// 构建 internal invoke request
	req := invokev1.NewInvokeMethodRequest(invokeMethodName).WithHTTPExtension(verb, reqCtx.QueryArgs().String())
	req.WithRawData(reqCtx.Request.Body(), string(reqCtx.Request.Header.ContentType()))
	// Save headers to internal metadata
	req.WithFastHTTPHeaders(&reqCtx.Request.Header)

  // 发出请求 
	resp, err := a.directMessaging.Invoke(reqCtx, targetID, req)
	// err does not represent user application response
	if err != nil {
		msg := NewErrorResponse("ERR_DIRECT_INVOKE", err.Error())
		respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
		return
	}

	invokev1.InternalMetadataToHTTPHeader(reqCtx, resp.Headers(), reqCtx.Response.Header.Set)
	contentType, body := resp.RawData()
	reqCtx.Response.Header.SetContentType(contentType)

	// 构建 response
	statusCode := int(resp.Status().Code)
	if !resp.IsHTTPResponse() {
		statusCode = invokev1.HTTPStatusFromCode(codes.Code(statusCode))
	}

	respond(reqCtx, statusCode, body)
}
```

发送请求的具体实现代码在 `pkg/messaging/direct_messaging.go` 中：

```go
func (d *directMessaging) Invoke(ctx context.Context, targetAppID string, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	if targetAppID == d.appID {
		return d.invokeLocal(ctx, req)
	}
	return d.invokeWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, targetAppID, d.invokeRemote, req)
}
```

后面的具体实现就和 gRPC API server / grpc dapr service 的实现一致了。

