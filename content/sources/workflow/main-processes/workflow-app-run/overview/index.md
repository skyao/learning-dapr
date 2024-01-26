---
title: "workflow app run流程概述"
linkTitle: "概述"
weight: 10
date: 2021-02-24
description: >
  workflow app中workflow runtime运行的源码概述
---

上一章看到 workflow runtime start 之后，就会启动任务处理的流程。

代码实现在 durabletask java sdk 中的 DurableTaskGrpcWorker 类的 startAndBlock()方法中。

这是最关键的代码。

先构建两个 executor，负责执行 Orchestration task 和 activity task：

```java
        TaskOrchestrationExecutor taskOrchestrationExecutor = new TaskOrchestrationExecutor(
                this.orchestrationFactories,
                this.dataConverter,
                this.maximumTimerInterval,
                logger);
        TaskActivityExecutor taskActivityExecutor = new TaskActivityExecutor(
                this.activityFactories,
                this.dataConverter,
                logger);
```

传入的参数有 orchestrationFactories 和 taskActivityExecutor，之前构建时注册的信息都保存在这里面。

## 获取工作任务

然后就是一个无限循环，在循环中调用 sidecarClient.getWorkItems(), 针对返回的 workitem stream，还有一个无限循环。而且如果遇到  StatusRuntimeException ，还会sleep之后继续。

```java
while (true) {
  try {
      GetWorkItemsRequest getWorkItemsRequest = GetWorkItemsRequest.newBuilder().build();
      Iterator<WorkItem> workItemStream = this.sidecarClient.getWorkItems(getWorkItemsRequest);
      while (workItemStream.hasNext()) {
        ......
      }
  } catch(StatusRuntimeException e){
    ......
    // Retry after 5 seconds
    try {
        Thread.sleep(5000);
    } catch (InterruptedException ex) {
        break;
    }
    }
}
```

work items 的类型只有两种 orchestrator 和 activity：

```java
while (workItemStream.hasNext()) {
    WorkItem workItem = workItemStream.next();
    RequestCase requestType = workItem.getRequestCase();
    if (requestType == RequestCase.ORCHESTRATORREQUEST) {
        ......
    } else if (requestType == RequestCase.ACTIVITYREQUEST) {
        ......
    } else {
        logger.log(Level.WARNING, "Received and dropped an unknown '{0}' work-item from the sidecar.", requestType);
    }
}
```

## 执行 orchestrator task

通过 taskOrchestrationExecutor 执行 orchestrator task，然后将结果返回给到 dapr sidecar。

```java
OrchestratorRequest orchestratorRequest = workItem.getOrchestratorRequest();

TaskOrchestratorResult taskOrchestratorResult = taskOrchestrationExecutor.execute(
        orchestratorRequest.getPastEventsList(),
        orchestratorRequest.getNewEventsList());

OrchestratorResponse response = OrchestratorResponse.newBuilder()
        .setInstanceId(orchestratorRequest.getInstanceId())
        .addAllActions(taskOrchestratorResult.getActions())
        .setCustomStatus(StringValue.of(taskOrchestratorResult.getCustomStatus()))
        .build();

this.sidecarClient.completeOrchestratorTask(response);
```

> 备注：比较奇怪的是这里为什么不用 grpc 双向 stream 来获取任务和返回任务执行结果，而是通过另外一个 completeOrchestratorTask() 方法来发起请求。

## 执行 avtivity task

类似的，通过 taskActivityExecutor 执行 avtivity task，然后将结果返回给到 dapr sidecar。

```java
ActivityRequest activityRequest = workItem.getActivityRequest();

String output = null;
TaskFailureDetails failureDetails = null;
try {
    output = taskActivityExecutor.execute(
        activityRequest.getName(),
        activityRequest.getInput().getValue(),
        activityRequest.getTaskId());
} catch (Throwable e) {
    failureDetails = TaskFailureDetails.newBuilder()
        .setErrorType(e.getClass().getName())
        .setErrorMessage(e.getMessage())
        .setStackTrace(StringValue.of(FailureDetails.getFullStackTrace(e)))
        .build();
}

ActivityResponse.Builder responseBuilder = ActivityResponse.newBuilder()
        .setInstanceId(activityRequest.getOrchestrationInstance().getInstanceId())
        .setTaskId(activityRequest.getTaskId());

if (output != null) {
    responseBuilder.setResult(StringValue.of(output));
}

if (failureDetails != null) {
    responseBuilder.setFailureDetails(failureDetails);
}

this.sidecarClient.completeActivityTask(responseBuilder.build());
```


