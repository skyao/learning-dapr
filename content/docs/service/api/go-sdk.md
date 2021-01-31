---
title: "Dapr服务调用的go sdk定义"
linkTitle: "go sdk定义"
weight: 525
date: 2021-01-29
description: >
  Dapr服务调用的go sdk定义
---

### go sdk使用案例

https://github.com/dapr/go-sdk 

要在另一个使用Dapr sidecar运行的服务上调用特定的方法，Dapr客户端提供了两个选项。

调用一个没有任何数据的服务：

```go
resp, err = client.InvokeService(ctx, "service-name", "method-name") 
```

还有带数据调用服务：

```
content := &DataContent{
    ContentType: "application/json",
    Data:        []byte(`{ "id": "a123", "value": "demo", "valid": true }`)
}

resp, err := client.InvokeServiceWithContent(ctx, "service-name", "method-name", content)
```

### go sdk提供的API

https://github.com/dapr/go-sdk/blob/d6de57c71a1d3c7ce3a3b81385609dfba18a1a18/client/invoke.go

go sdk在 client 上封装了两个方法用于服务调用，InvokeService方法用来发送不带数据的请求：

```go
// InvokeService invokes service without raw data ([]byte).
func (c *GRPCClient) InvokeService(ctx context.Context, serviceID, method string) (out []byte, err error) {
...
}
```

InvokeServiceWithContent方法用来发现带数据的请求：

```go
// InvokeServiceWithContent invokes service without content (data + content type).
func (c *GRPCClient) InvokeServiceWithContent(ctx context.Context, serviceID, method string, content *DataContent) (out []byte, err error) {
......
}
```

DataContent 的定义：

```go
// DataContent the service invocation content
type DataContent struct {
	// Data is the input data
	Data []byte
	// ContentType is the type of the data content
	ContentType string
}
```



