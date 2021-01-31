---
title: "状态管理API的go client定义"
linkTitle: "go client定义"
weight: 824
date: 2021-01-31
description: >
  Dapr状态管理API的golang生成代码
---


### DaprClient

https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/pkg/proto/runtime/v1/dapr.pb.go

这是根据proto生成的go代码

```go
type DaprClient interface {
	// Gets the state for a specific key.
	GetState(ctx context.Context, in *GetStateRequest, opts ...grpc.CallOption) (*GetStateResponse, error)
	// Gets a bulk of state items for a list of keys
	GetBulkState(ctx context.Context, in *GetBulkStateRequest, opts ...grpc.CallOption) (*GetBulkStateResponse, error)
	// Saves the state for a specific key.
	SaveState(ctx context.Context, in *SaveStateRequest, opts ...grpc.CallOption) (*empty.Empty, error)
	// Deletes the state for a specific key.
	DeleteState(ctx context.Context, in *DeleteStateRequest, opts ...grpc.CallOption) (*empty.Empty, error)
	// Executes transactions for a specified store
	ExecuteStateTransaction(ctx context.Context, in *ExecuteStateTransactionRequest, opts ...grpc.CallOption) (*empty.Empty, error)
	......
}
```
### Get State

以 Get State 为例看 DaprClient 的实现：

```go
func (c *daprClient) GetState(ctx context.Context, in *GetStateRequest, opts ...grpc.CallOption) (*GetStateResponse, error) {
	out := new(GetStateResponse)
	err := c.cc.Invoke(ctx, "/dapr.proto.runtime.v1.Dapr/GetState", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

只是简单调用远程方法。

