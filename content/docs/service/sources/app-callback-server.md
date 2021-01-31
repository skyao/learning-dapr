---
title: "服务调用AppCallbackServer的实现"
linkTitle: "AppCallbackServer的实现"
weight: 546
date: 2021-01-31
description: >
  Dapr服务调用AppCallbackServer的实现
---

`pkg/proto/runtime/v1/appcallback.pb.go` 中的 OnInvoke 方法：

```go
// AppCallbackServer is the server API for AppCallback service.
type AppCallbackServer interface {
	// Invokes service method with InvokeRequest.
	OnInvoke(context.Context, *v1.InvokeRequest) (*v1.InvokeResponse, error)
}
```

## 实现

AppCallbackServer 的实现在各个不同语言的 sdk 里面。

### go-sdk实现

实现在 go-sdk 的 `service/grpc/invoke.go` 文件的 OnInvoke方法：

```go
func (s *Server) OnInvoke(ctx context.Context, in *cpb.InvokeRequest) (*cpb.InvokeResponse, error) {
	if in == nil {
		return nil, errors.New("nil invoke request")
	}
	if fn, ok := s.invokeHandlers[in.Method]; ok {
		e := &cc.InvocationEvent{}
		if in != nil {
			e.ContentType = in.ContentType

			if in.Data != nil {
				e.Data = in.Data.Value
				e.DataTypeURL = in.Data.TypeUrl
			}

			if in.HttpExtension != nil {
				e.Verb = in.HttpExtension.Verb.String()
				e.QueryString = in.HttpExtension.Querystring
			}
		}

		ct, er := fn(ctx, e)
		if er != nil {
			return nil, errors.Wrap(er, "error executing handler")
		}

		if ct == nil {
			return &cpb.InvokeResponse{}, nil
		}

		return &cpb.InvokeResponse{
			ContentType: ct.ContentType,
			Data: &any.Any{
				Value:   ct.Data,
				TypeUrl: ct.DataTypeURL,
			},
		}, nil
	}
	return nil, fmt.Errorf("method not implemented: %s", in.Method)
}
```

其中 `s.invokeHandlers[in.Method]` 是要求找到处理请求的对应方法（由参数method制定）。这些方法通过方法名进行映射，增加方法名到hanlder的方法是：

```go
func (s *Server) AddServiceInvocationHandler(method string, fn func(ctx context.Context, in *cc.InvocationEvent) (our *cc.Content, err error)) error {
	if method == "" {
		return fmt.Errorf("servie name required")
	}
	s.invokeHandlers[method] = fn
	return nil
}
```

## 用户实现

用户在开发支持 dapr 的服务器端应用时，需要在应用中启动 dapr service server，然后添加各种 handler，包括 ServiceInvocationHandler，如下面这个例子（go-sdk下的 `example/serving/grpc/main.go` ）：

```go
func main() {
	// create a Dapr service server
	s, err := daprd.NewService(":50001")
	if err != nil {
		log.Fatalf("failed to start the server: %v", err)
	}

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






