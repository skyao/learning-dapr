---
title: "资源绑定API的go sdk"
linkTitle: "go sdk"
weight: 725
date: 2021-01-31
description: >
  Dapr的资源绑定API的go sdk
---


## Output Binding

### go sdk使用案例

https://github.com/dapr/go-sdk 

与Service类似，Dapr客户端提供了两种方法来调用Dapr定义的绑定上的操作。Dapr支持输入、输出和双向绑定。

对于简单的，只输出的绑定。

```go
in := &dapr.BindingInvocation{ Name: "binding-name", Operation: "operation-name" }
err = client.InvokeOutputBinding(ctx, in)
```

调用带有内容和元数据的方法：

```
in := &dapr.BindingInvocation{
    Name:      "binding-name",
    Operation: "operation-name",
    Data: []byte("hello"),
    Metadata: map[string]string{"k1": "v1", "k2": "v2"},
}

out, err := client.InvokeBinding(ctx, in)
```

### go sdk提供的API

/client/invoke.go

go sdk在 client 上封装了 InvokeOutputBinding 方法用于发起 output binding 调用：

```go
// InvokeOutputBinding invokes configured Dapr binding with data (allows nil).InvokeOutputBinding
// This method differs from InvokeBinding in that it doesn't expect any content being returned from the invoked method.
func (c *GRPCClient) InvokeOutputBinding(ctx context.Context, in *BindingInvocation) error {
	if _, err := c.InvokeBinding(ctx, in); err != nil {
		return errors.Wrap(err, "error invoking output binding")
	}
	return nil
}
```

InvokeServiceWithContent方法用来发现带数据的请求：

```go
// InvokeBinding invokes specific operation on the configured Dapr binding.
// This method covers input, output, and bi-directional bindings.
func (c *GRPCClient) InvokeBinding(ctx context.Context, in *BindingInvocation) (out *BindingEvent, err error) {
	if in == nil {
		return nil, errors.New("binding invocation required")
	}
	if in.Name == "" {
		return nil, errors.New("binding invocation name required")
	}
	if in.Operation == "" {
		return nil, errors.New("binding invocation operation required")
	}

	req := &pb.InvokeBindingRequest{
		Name:      in.Name,
		Operation: in.Operation,
		Data:      in.Data,
		Metadata:  in.Metadata,
	}

	resp, err := c.protoClient.InvokeBinding(authContext(ctx), req)
	if err != nil {
		return nil, errors.Wrapf(err, "error invoking binding %s/%s", in.Name, in.Operation)
	}

	out = &BindingEvent{}

	if resp != nil {
		out.Data = resp.Data
		out.Metadata = resp.Metadata
	}

	return
}
```

BindingInvocation 的定义：

```go
// BindingInvocation represents binding invocation request
type BindingInvocation struct {
	// Name is name of binding to invoke.
	Name string
	// Operation is the name of the operation type for the binding to invoke
	Operation string
	// Data is the input bindings sent
	Data []byte
	// Metadata is the input binding metadata
	Metadata map[string]string
}
```

和根据 proto 生成的 InvokeBindingRequest 是完全一样的，除了去除了生成的 state/sizeCache/unknownFields 等字段。

## Input Binding

TODO