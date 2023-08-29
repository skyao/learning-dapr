---
title: "workflow定义和操作方法"
linkTitle: "workflow定义"
weight: 100
date: 2021-03-03
description: >
  workflow的定义和操作方法的具体内容
---

代码量比较少，就放在一起看吧。

## 接口定义

### workflow 接口

workflow 接口定义了 workflow 上要履行的操作：

```go
var ErrNotImplemented = errors.New("this component doesn't implement the current API operation")

type Workflow interface {
	Init(metadata Metadata) error
	Start(ctx context.Context, req *StartRequest) (*StartResponse, error)
	Terminate(ctx context.Context, req *TerminateRequest) error
	Get(ctx context.Context, req *GetRequest) (*StateResponse, error)
	RaiseEvent(ctx context.Context, req *RaiseEventRequest) error
	Purge(ctx context.Context, req *PurgeRequest) error
	Pause(ctx context.Context, req *PauseRequest) error
	Resume(ctx context.Context, req *ResumeRequest) error
}
```

其中 Init 是初始化 workflow 实现。

Start / Terminate / Pause / Resume 是 workflow 的生命周期管理。

如果没有实现上述操作，则需要返回错误，而错误信息在 ErrNotImplemented 中有统一给出。

## 操作

### init 操作

通过 metadata 进行初始化，和其他组件类似：

```go
type Workflow interface {
	Init(metadata Metadata) error
	......
}

type Metadata struct {
	metadata.Base `json:",inline"`
}
```

### Start 操作

start 操作用来开始一个工作流：

```go
type Workflow interface {
	Start(ctx context.Context, req *StartRequest) (*StartResponse, error)
	......
}

// StartRequest is the struct describing a start workflow request.
type StartRequest struct {
	InstanceID    string            `json:"instanceID"`
	Options       map[string]string `json:"options"`
	WorkflowName  string            `json:"workflowName"`
	WorkflowInput []byte            `json:"workflowInput"`
}

type StartResponse struct {
	InstanceID string `json:"instanceID"`
}
```

start 操作的请求参数是：

- InstanceID：
- Options：map[string]string
- WorkflowName：
- WorkflowInput： []byte


start 操作的响应参数是：

- InstanceID：

### Terminate 操作

Terminate 操作用来终止一个 workflow：

```go
type Workflow interface {
	Terminate(ctx context.Context, req *TerminateRequest) error
}

type TerminateRequest struct {
	InstanceID string `json:"instanceID"`
}
```

start 操作的请求只需要传递一个 InstanceID 参数。

### Get 操作

Get 操作用来或者一个工作流实例的状态：

```go
type Workflow interface {
	Get(ctx context.Context, req *GetRequest) (*StateResponse, error)
	......
}

type GetRequest struct {
	InstanceID string `json:"instanceID"`
}

type StateResponse struct {
	Workflow *WorkflowState `json:"workflow"`
}

type WorkflowState struct {
	InstanceID    string            `json:"instanceID"`
	WorkflowName  string            `json:"workflowName"`
	CreatedAt     time.Time         `json:"startedAt"`
	LastUpdatedAt time.Time         `json:"lastUpdatedAt"`
	RuntimeStatus string            `json:"runtimeStatus"`
	Properties    map[string]string `json:"properties"`
}
```

Get 操作的请求只需要传递一个 InstanceID 参数。


Get 操作的响应参数是 WorkflowState，字段有：

- InstanceID：
- WorkflowName：
- CreatedAt
- LastUpdatedAt
- RuntimeStatus
- Properties



### Purge 操作

Purge 操作用来终止一个 workflow：

```go
type Workflow interface {
	Purge(ctx context.Context, req *PurgeRequest) error
}

type PurgeRequest struct {
	InstanceID string `json:"instanceID"`
}
```

Purge 操作的请求只需要传递一个 InstanceID 参数。

### Pause 操作

Pause 操作用来暂停一个 workflow：

```go
type Workflow interface {
	Pause(ctx context.Context, req *PauseRequest) error
}

type PauseRequest struct {
	InstanceID string `json:"instanceID"`
}
```

Pause 操作的请求只需要传递一个 InstanceID 参数。

### Resume 操作

Resume 操作用来继续一个 workflow：

```go
type Workflow interface {
	Resume(ctx context.Context, req *ResumeRequest) error
}

type ResumeRequest struct {
	InstanceID string `json:"instanceID"`
}
```

Resume 操作的请求只需要传递一个 InstanceID 参数。