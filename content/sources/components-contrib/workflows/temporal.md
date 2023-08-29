---
title: "temporal集成"
linkTitle: "temporal"
weight: 200
date: 2021-03-03
description: >
  temporal集成的实现
---

## workflow 定义

TemporalWF 结构体包含 temporal 的 client：

```go
type TemporalWF struct {
	client client.Client
	logger logger.Logger
}
```

temporalMetadata 结构体定义 metadata：


```go
type temporalMetadata struct {
	Identity  string `json:"identity" mapstructure:"identity"`
	HostPort  string `json:"hostport" mapstructure:"hostport"`
	Namespace string `json:"namespace" mapstructure:"namespace"`
}
```

## 创建workflow

### NewTemporalWorkflow()方法

```go
// NewTemporalWorkflow returns a new workflow.
func NewTemporalWorkflow(logger logger.Logger) workflows.Workflow {
	s := &TemporalWF{
		logger: logger,
	}
	return s
}
```

### Init()方法


```go
func (c *TemporalWF) Init(metadata workflows.Metadata) error {
	c.logger.Debugf("Temporal init start")
	m, err := c.parseMetadata(metadata)
	if err != nil {
		return err
	}
	cOpt := client.Options{}
	if m.HostPort != "" {
		cOpt.HostPort = m.HostPort
	}
	if m.Identity != "" {
		cOpt.Identity = m.Identity
	}
	if m.Namespace != "" {
		cOpt.Namespace = m.Namespace
	}
	// Create the workflow client
	newClient, err := client.Dial(cOpt)
	if err != nil {
		return err
	}
	c.client = newClient

	return nil
}

func (c *TemporalWF) parseMetadata(meta workflows.Metadata) (*temporalMetadata, error) {
	var m temporalMetadata
	err := metadata.DecodeMetadata(meta.Properties, &m)
	return &m, err
}
```

## workflow操作

### Start


```go
func (c *TemporalWF) Start(ctx context.Context, req *workflows.StartRequest) (*workflows.StartResponse, error) {
	c.logger.Debugf("starting workflow")

	if len(req.Options) == 0 {
		c.logger.Debugf("no options provided")
		return nil, errors.New("no options provided. At the very least, a task queue is needed")
	}

	if _, ok := req.Options["task_queue"]; !ok {
		c.logger.Debugf("no task queue provided")
		return nil, errors.New("no task queue provided")
	}
	taskQ := req.Options["task_queue"]

	opt := client.StartWorkflowOptions{ID: req.InstanceID, TaskQueue: taskQ}

	var inputArgs interface{}
	if err := decodeInputData(req.WorkflowInput, &inputArgs); err != nil {
		return nil, fmt.Errorf("error decoding workflow input data: %w", err)
	}

	run, err := c.client.ExecuteWorkflow(ctx, opt, req.WorkflowName, inputArgs)
	if err != nil {
		return nil, fmt.Errorf("error executing workflow: %w", err)
	}
	wfStruct := workflows.StartResponse{InstanceID: run.GetID()}
	return &wfStruct, nil
}
```

代码和 temporal 的牵连还是很重的，WorkflowInput 相当于透传给了 temporal ，dapr 对此没有做任何的抽象和封装，只是简单透传。

### Terminate

```go
func (c *TemporalWF) Terminate(ctx context.Context, req *workflows.TerminateRequest) error {
	c.logger.Debugf("terminating workflow")

	err := c.client.TerminateWorkflow(ctx, req.InstanceID, "", "")
	if err != nil {
		return fmt.Errorf("error terminating workflow: %w", err)
	}
	return nil
}
```

