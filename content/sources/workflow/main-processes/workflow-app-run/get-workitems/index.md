---
title: "获取工作任务"
linkTitle: "获取工作任务"
weight: 20
date: 2021-02-24
description: >
  workflow runtime运行时获取工作任务的源码概述
---

## 获取工作任务的调用代码

DurableTaskGrpcWorker 会调用 sidecarClient.getWorkItems() 来获取工作任务。

```java
private final TaskHubSidecarServiceBlockingStub sidecarClient;

while (true) {
  try {
      GetWorkItemsRequest getWorkItemsRequest = GetWorkItemsRequest.newBuilder().build();
      Iterator<WorkItem> workItemStream = this.sidecarClient.getWorkItems(getWorkItemsRequest);
      while (workItemStream.hasNext()) {
        ......
      }
  } catch{}
}
```

## 代码实现

### proto 定义

TaskHubSidecarServiceBlockingStub 是根据 protobuf 文件生成的 grpc 代码，其 protobuf 定义在`submodules/durabletask-protobuf/protos/orchestrator_service.proto` 文件中。 

```protobuf
service TaskHubSidecarService {
    ......
    rpc GetWorkItems(GetWorkItemsRequest) returns (stream WorkItem);
    ......
}
```

GetWorkItemsRequest 和 WorkItem 的消息定义为：

```protobuf
message GetWorkItemsRequest {
    // No parameters currently
}

message WorkItem {
    oneof request {
        OrchestratorRequest orchestratorRequest = 1;
        ActivityRequest activityRequest = 2;
    }
}
```

WorkItem 可能是 OrchestratorRequest 或者 ActivityRequest 。

#### OrchestratorRequest

```protobuf
message OrchestratorRequest {
    string instanceId = 1;
    google.protobuf.StringValue executionId = 2;
    repeated HistoryEvent pastEvents = 3;
    repeated HistoryEvent newEvents = 4;
}
```

#### ActivityRequest

```protobuf
message ActivityRequest {
    string name = 1;
    google.protobuf.StringValue version = 2;
    google.protobuf.StringValue input = 3;
    OrchestrationInstance orchestrationInstance = 4;
    int32 taskId = 5;
}
```

#### HistoryEvent

```protobuf
message HistoryEvent {
    int32 eventId = 1;
    google.protobuf.Timestamp timestamp = 2;
    oneof eventType {
        ExecutionStartedEvent executionStarted = 3;
        ExecutionCompletedEvent executionCompleted = 4;
        ExecutionTerminatedEvent executionTerminated = 5;
        TaskScheduledEvent taskScheduled = 6;
        TaskCompletedEvent taskCompleted = 7;
        TaskFailedEvent taskFailed = 8;
        SubOrchestrationInstanceCreatedEvent subOrchestrationInstanceCreated = 9;
        SubOrchestrationInstanceCompletedEvent subOrchestrationInstanceCompleted = 10;
        SubOrchestrationInstanceFailedEvent subOrchestrationInstanceFailed = 11;
        TimerCreatedEvent timerCreated = 12;
        TimerFiredEvent timerFired = 13;
        OrchestratorStartedEvent orchestratorStarted = 14;
        OrchestratorCompletedEvent orchestratorCompleted = 15;
        EventSentEvent eventSent = 16;
        EventRaisedEvent eventRaised = 17;
        GenericEvent genericEvent = 18;
        HistoryStateEvent historyState = 19;
        ContinueAsNewEvent continueAsNew = 20;
        ExecutionSuspendedEvent executionSuspended = 21;
        ExecutionResumedEvent executionResumed = 22;
    }
}
```

### worker 调用

workflow app 中通过调用 sidecarClient.getWorkItems() 方法来获取 work items。

```java
Iterator<WorkItem> workItemStream = this.sidecarClient.getWorkItems(getWorkItemsRequest);
```

这里面就是 grpc stub 的生成代码，不细看

### TaskHubSidecarService 服务器实现

TaskHubSidecarService 这个 protobuf 定义的 grpc service 的服务器端，代码实现在 durabletask-go 仓库中。

protobuf 生成的 grpc stub 的类在这里：

- internal/protos/orchestrator_service_grpc.pb.go
- internal/protos/orchestrator_service.pb.go

服务器端代码实现在 `backend/executor.go` 中：

```go
// GetWorkItems implements protos.TaskHubSidecarServiceServer
func (g *grpcExecutor) GetWorkItems(req *protos.GetWorkItemsRequest, stream protos.TaskHubSidecarService_GetWorkItemsServer) error {
    ......

	// The worker client invokes this method, which streams back work-items as they arrive.
	for {
		select {
		case <-stream.Context().Done():
			g.logger.Infof("work item stream closed")
			return nil
		case wi := <-g.workItemQueue:
			if err := stream.Send(wi); err != nil {
				return err
			}
		case <-g.streamShutdownChan:
			return errShuttingDown
		}
	}
}
```

所以返回给客户端调用的 work item stream 的数据来自 g.workItemQueue 

```go
type grpcExecutor struct {
    ......
	workItemQueue        chan *protos.WorkItem
}
```

### workItemQueue 的实现逻辑

workItemQueue 在 grpcExecutor 中定义：

```protobuf
type grpcExecutor struct {
	workItemQueue        chan *protos.WorkItem
    ......
}
```

grpcExecutor 在 NewGrpcExecutor() 方法中构建：

```protobuf
// NewGrpcExecutor returns the Executor object and a method to invoke to register the gRPC server in the executor.
func NewGrpcExecutor(be Backend, logger Logger, opts ...grpcExecutorOptions) (executor Executor, registerServerFn func(grpcServer grpc.ServiceRegistrar)) {
	grpcExecutor := &grpcExecutor{
		workItemQueue:        make(chan *protos.WorkItem),
		backend:              be,
		logger:               logger,
		pendingOrchestrators: &sync.Map{},
		pendingActivities:    &sync.Map{},
	}

    ......
}
```

将数据写入 workItemQueue 的地方有两个：

1. ExecuteOrchestrator()

    ```go
    func (executor *grpcExecutor) ExecuteOrchestrator(......) {
        ......
            workItem := &protos.WorkItem{
            Request: &protos.WorkItem_OrchestratorRequest{
                OrchestratorRequest: &protos.OrchestratorRequest{
                    InstanceId:  string(iid),
                    ExecutionId: nil,
                    PastEvents:  oldEvents,
                    NewEvents:   newEvents,
                },
            },
        }

        executor.workItemQueue <- workItem:
    }
    ```

2. ExecuteActivity()

    ```go
    func (executor *grpcExecutor) ExecuteActivity(......) {
        workItem := &protos.WorkItem{
		Request: &protos.WorkItem_ActivityRequest{
			ActivityRequest: &protos.ActivityRequest{
				Name:                  task.Name,
				Version:               task.Version,
				Input:                 task.Input,
				OrchestrationInstance: &protos.OrchestrationInstance{InstanceId: string(iid)},
				TaskId:                e.EventId,
			},
		},

        executor.workItemQueue <- workItem:
	}
    ```

继续跟踪看 ExecuteOrchestrator() 和 ExecuteActivity() 方法是被谁调用的，这个细节在下一节中。

### 小结

获取工作任务的任务源头在 dapr sidecar，代码实现在 durabletask-go 项目的 `backend/executor.go` 中。