---
title: "状态管理API的golang生成代码"
linkTitle: "golang生成代码"
weight: 823
date: 2021-01-31
description: >
  Dapr状态管理API的golang生成代码
---

从proto api定义文件生成的golang代码，被存放在dapr项目的 ` pkg/proto/` 目录下。

### grpc服务定义

DaprServer 是 dapr 服务的服务器端API定义，包含多个 state 相关的方法：

```go
// DaprServer is the server API for Dapr service.
type DaprServer interface {
	// Gets the state for a specific key.
	GetState(context.Context, *GetStateRequest) (*GetStateResponse, error)
	// Gets a bulk of state items for a list of keys
	GetBulkState(context.Context, *GetBulkStateRequest) (*GetBulkStateResponse, error)
	// Saves the state for a specific key.
	SaveState(context.Context, *SaveStateRequest) (*empty.Empty, error)
	// Deletes the state for a specific key.
	DeleteState(context.Context, *DeleteStateRequest) (*empty.Empty, error)
	// Executes transactions for a specified store
	ExecuteStateTransaction(context.Context, *ExecuteStateTransactionRequest) (*empty.Empty, error)
   ......
}
```

