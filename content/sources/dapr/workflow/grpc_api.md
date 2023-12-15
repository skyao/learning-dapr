---
title: "workflow gRPC API"
linkTitle: "gRPC API实现"
weight: 30
date: 2021-01-31
description: >
  Dapr workflow的gRPC API实现
---

## proto 定义

`dapr/proto/runtime/v1/dapr.proto` 

```protobuf
service Dapr {
  // Starts a new instance of a workflow
  rpc StartWorkflowAlpha1 (StartWorkflowRequest) returns (StartWorkflowResponse) {}

  // Gets details about a started workflow instance
  rpc GetWorkflowAlpha1 (GetWorkflowRequest) returns (GetWorkflowResponse) {}

  // Purge Workflow
  rpc PurgeWorkflowAlpha1 (PurgeWorkflowRequest) returns (google.protobuf.Empty) {}

  // Terminates a running workflow instance
  rpc TerminateWorkflowAlpha1 (TerminateWorkflowRequest) returns (google.protobuf.Empty) {}

  // Pauses a running workflow instance
  rpc PauseWorkflowAlpha1 (PauseWorkflowRequest) returns (google.protobuf.Empty) {}

  // Resumes a paused workflow instance
  rpc ResumeWorkflowAlpha1 (ResumeWorkflowRequest) returns (google.protobuf.Empty) {}

  // Raise an event to a running workflow instance
  rpc RaiseEventWorkflowAlpha1 (RaiseEventWorkflowRequest) returns (google.protobuf.Empty) {}
}
```

workflow 没有 sidecar 往应用方向发请求的场景，也就是没有 appcallback 。

### 生成的 go 代码

`pkg/proto/runtime/v1` 下存放的是根据 proto 生成的 go 代码

比如 `pkg/proto/runtime/v1/dapr_grpc.pb.go`