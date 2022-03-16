---
type: docs
title: "服务调用的客户端sdk封装"
linkTitle: "客户端sdk封装"
weight: 230
date: 2021-01-31
description: >
  Dapr服务调用的客户端sdk封装
---


## Go sdk实现

go sdk 将 dapr grpc service 中定义的 InvokeService 方法封装为  InvokeService 方法和 InvokeServiceWithContent：

```go
// InvokeService invokes service without raw data ([]byte).
func (c *GRPCClient) InvokeService(ctx context.Context, serviceID, method string) (out []byte, err error) {
	if serviceID == "" {
		return nil, errors.New("nil serviceID")
	}
	if method == "" {
		return nil, errors.New("nil method")
	}
	req := &pb.InvokeServiceRequest{
		Id: serviceID,
		Message: &v1.InvokeRequest{
			Method: method,
			HttpExtension: &v1.HTTPExtension{
				Verb: v1.HTTPExtension_POST,
			},
		},
	}
	return c.invokeServiceWithRequest(ctx, req)
}
```
InvokeServiceWithContent方法用来发现带数据的请求：

```go
// InvokeServiceWithContent invokes service without content (data + content type).
func (c *GRPCClient) InvokeServiceWithContent(ctx context.Context, serviceID, method string, content *DataContent) (out []byte, err error) {
	if serviceID == "" {
		return nil, errors.New("serviceID is required")
	}
	if method == "" {
		return nil, errors.New("method name is required")
	}
	if content == nil {
		return nil, errors.New("content required")
	}

	req := &pb.InvokeServiceRequest{
		Id: serviceID,
		Message: &v1.InvokeRequest{
			Method:      method,
			Data:        &anypb.Any{Value: content.Data},
			ContentType: content.ContentType,
			HttpExtension: &v1.HTTPExtension{
				Verb: v1.HTTPExtension_POST,
			},
		},
	}

	return c.invokeServiceWithRequest(ctx, req)
}
```

从实现上分析都只是实现了对 InvokeRequest 对象的组装，最后代码实现在 invokeServiceWithRequest 方法中：

```go
func (c *GRPCClient) invokeServiceWithRequest(ctx context.Context, req *pb.InvokeServiceRequest) (out []byte, err error) {
	if req == nil {
		return nil, errors.New("nil request")
	}

   // 调用proto定义的 InvokeService 方法
	resp, err := c.protoClient.InvokeService(authContext(ctx), req)
	if err != nil {
		return nil, errors.Wrap(err, "error invoking service")
	}

	// allow for service to not return any value
	if resp != nil && resp.GetData() != nil {
		out = resp.GetData().Value
		return
	}

	out = nil
	return
}
```

## Java SDK 实现

在业务代码中使用 service invoke 功能的示例可参考文件 `java-sdk/examples/src/main/java/io/dapr/examples/invoke/http/InvokeClient.java`，代码示意如下：

```java
DaprClient client = (new DaprClientBuilder()).build();
byte[] response = client.invokeMethod(SERVICE_APP_ID, "say", message, HttpExtension.POST, null,
            byte[].class).block();
```







### 分析

实现了从客户端SDK API到发出 grpc service 远程调用请求给 dapr runtime的功能。

代码实现很简单，非常薄的一点点封装逻辑。

