---
title: "DaprWorkflowClient代码实现"
linkTitle: "DaprWorkflowClient"
weight: 10
date: 2022-07-24
description: >
  DaprWorkflowClient 的代码实现
---

## 定义和创建

### 类定义

DaprWorkflowClient 定义管理 Dapr 工作流实例的客户端操作。

**注意这里是 “管理” ！**

```java
import com.microsoft.durabletask.DurableTaskClient;

public class DaprWorkflowClient implements AutoCloseable {

  DurableTaskClient innerClient;
  ManagedChannel grpcChannel;
    
  public DaprWorkflowClient() {
    this(NetworkUtils.buildGrpcManagedChannel());
  }
    
  private DaprWorkflowClient(ManagedChannel grpcChannel) {
    this(createDurableTaskClient(grpcChannel), grpcChannel);
  }
    
  private DaprWorkflowClient(DurableTaskClient innerClient, ManagedChannel grpcChannel) {
    this.innerClient = innerClient;
    this.grpcChannel = grpcChannel;
  }
```

实现上依然是包装 durabletask 的 DurableTaskClient ， 而 durabletask 的 DurableTaskClient 在创建时需要传入一个 grpcChannel。

关键点在于这个 grpcChannel 的创建，可以从外部传入，如果没有传入则可以通过 NetworkUtils.buildGrpcManagedChannel() 方法进行创建。

### grpcChannel 的创建

实现和之前 WorkflowRuntimeBuilder 中的一致，都是调用 `NetworkUtils.buildGrpcManagedChannel()` 方法。

`NetworkUtils.buildGrpcManagedChannel()` 方法在 dapr java sdk 中一共有3处调用：

1. WorkflowRuntimeBuilder：

    ```java
      public WorkflowRuntimeBuilder() {
        this.builder = new DurableTaskGrpcWorkerBuilder().grpcChannel(NetworkUtils.buildGrpcManagedChannel());
      }
    ```

2. DaprWorkflowClient：

    ```java
      public DaprWorkflowClient() {
        this(NetworkUtils.buildGrpcManagedChannel());
      }
    ```

3. DaprClientBuilder

    ```java
    final ManagedChannel channel = NetworkUtils.buildGrpcManagedChannel();
    ```

### DurableTaskClient 的创建

DurableTaskClient 的创建是简单的调用 durabletask 的 DurableTaskGrpcClientBuilder 来实现的：

```java
import com.microsoft.durabletask.DurableTaskGrpcClientBuilder;

private static DurableTaskClient createDurableTaskClient(ManagedChannel grpcChannel) {
    return new DurableTaskGrpcClientBuilder()
        .grpcChannel(grpcChannel)
        .build();
  }
```

### close() 方法

close() 方法用于关闭 DaprWorkflowClient，内部实现为关闭包装的 durabletask 的 DurableTaskClient 以及创建时传入的 grpcChannel：

```java
  public void close() throws InterruptedException {
    try {
      if (this.innerClient != null) {
        this.innerClient.close();
        this.innerClient = null;
      }
    } finally {
      if (this.grpcChannel != null && !this.grpcChannel.isShutdown()) {
        this.grpcChannel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
        this.grpcChannel = null;
      }
    }
  }
}
```



## 操作 workflow instance

### scheduleNewWorkflow() 方法

scheduleNewWorkflow() 方法调度一个新的 workflow ，即创建并开始一个新的 workflow instance，这个方法返回 workflow  instance id：

```java
package io.dapr.workflows.client;  

public <T extends Workflow> String scheduleNewWorkflow(Class<T> clazz) {
    return this.innerClient.scheduleNewOrchestrationInstance(clazz.getCanonicalName());
  }

  public <T extends Workflow> String scheduleNewWorkflow(Class<T> clazz, Object input) {
    return this.innerClient.scheduleNewOrchestrationInstance(clazz.getCanonicalName(), input);
  }

  public <T extends Workflow> String scheduleNewWorkflow(Class<T> clazz, Object input, String instanceId) {
    return this.innerClient.scheduleNewOrchestrationInstance(clazz.getCanonicalName(), input, instanceId);
  }
```

实现完全代理给  durabletask 的 DurableTaskClient 。

### terminateWorkflow() 方法

terminateWorkflow() 方法终止一个 workflow instance 的执行，需要传入之前从 scheduleNewWorkflow() 方法中得到的 workflow  instance id。

```java
  public void terminateWorkflow(String workflowInstanceId, @Nullable Object output) {
    this.innerClient.terminate(workflowInstanceId, output);
  }
```

output 参数是可选的，用来传递被终止的 workflow instance 的输出。

### getInstanceState() 方法

getInstanceState() 方法获取 workflow instance 的状态，同样需要传入之前从 scheduleNewWorkflow() 方法中得到的 workflow  instance id：

```java
  @Nullable
  public WorkflowInstanceStatus getInstanceState(String instanceId, boolean getInputsAndOutputs) {
    OrchestrationMetadata metadata = this.innerClient.getInstanceMetadata(instanceId, getInputsAndOutputs);
    if (metadata == null) {
      return null;
    }
    return new WorkflowInstanceStatus(metadata);
  }
```

实现为调用 durabletask 的 DurableTaskClient 的 getInstanceMetadata() 方法来获取 OrchestrationMetadata，然后转换为 dapr 定义的 WorkflowInstanceStatus()。

这里的细节在 WorkflowInstanceStatus 类实现中展开。

### waitForInstanceStart() 方法

waitForInstanceStart() 方法等待 workflow instance 执行的开始：

```java
  @Nullable
  public WorkflowInstanceStatus waitForInstanceStart(String instanceId, Duration timeout, boolean getInputsAndOutputs)
      throws TimeoutException {

    OrchestrationMetadata metadata = this.innerClient.waitForInstanceStart(instanceId, timeout, getInputsAndOutputs);
    return metadata == null ? null : new WorkflowInstanceStatus(metadata);
  }
```

waitForInstanceStart() 方法的 javadoc 描述为：

> 等待工作流开始运行，并返回一个 WorkflowInstanceStatus 对象，该对象包含已启动实例的元数据，以及可选的输入、输出和自定义状态有效载荷。
>
> "已启动" 的工作流实例是指未处于 "Pending" 状态的任何实例。
>
> 如果调用该方法时工作流实例已在运行，该方法将立即返回。

### waitForInstanceCompletion() 方法

waitForInstanceCompletion() 方法等待 workflow instance 执行的完成：

```java
  @Nullable
  public WorkflowInstanceStatus waitForInstanceCompletion(String instanceId, Duration timeout,
                                                          boolean getInputsAndOutputs) throws TimeoutException {

    OrchestrationMetadata metadata =
        this.innerClient.waitForInstanceCompletion(instanceId, timeout, getInputsAndOutputs);
    return metadata == null ? null : new WorkflowInstanceStatus(metadata);
  }
```

waitForInstanceStart() 方法的 javadoc 描述为：

> 等待工作流完成，并返回一个包含已完成实例元数据的 WorkflowInstanceStatus 对象。
>
> "已完成" 的工作流实例是指处于终止状态之一的任何实例。例如，"Completed"、"Failed" 或 "Terminated" 状态。
>
> 工作流是长期运行的，可能需要数小时、数天或数月才能完成。工作流也可能是长久的，在这种情况下，除非终止，否则永远不会完成。在这种情况下，该调用可能会无限期阻塞，因此必须注意确保使用适当的超时。如果调用该方法时工作流实例已经完成，该方法将立即返回。

### purgeInstance() 方法

purgeInstance() 方法从工作流状态存储中清除工作流实例的状态：

```java
  public boolean purgeInstance(String workflowInstanceId) {
    PurgeResult result = this.innerClient.purgeInstance(workflowInstanceId);
    if (result != null) {
      return result.getDeletedInstanceCount() > 0;
    }
    return false;
  }
```

如果找到工作流状态并成功清除，则返回 true，否则返回 false。



### raiseEvent() 方法

raiseEvent() 方法向等待中的工作流实例发送事件通知消息：

```java
  public void raiseEvent(String workflowInstanceId, String eventName, Object eventPayload) {
    this.innerClient.raiseEvent(workflowInstanceId, eventName, eventPayload);
  }
```



## TaskHub的方法

这两个方法暂时还知道什么情况下用，暂时忽略。

```java
  public void createTaskHub(boolean recreateIfExists) {
    this.innerClient.createTaskHub(recreateIfExists);
  }

  public void deleteTaskHub() {
    this.innerClient.deleteTaskHub();
  }
```



