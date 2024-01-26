---
title: "构建workflowruntime的源码"
linkTitle: "构建workflow runtime"
weight: 20
date: 2021-02-24
description: >
  workflow app start流程中构建workflow runtime的源码
---

## 调用代码

workflow app 中构建 WorkflowRuntime 的典型使用代码如下：

```java
    // Register the OrderProcessingWorkflow and its activities with the builder.
    WorkflowRuntimeBuilder builder = new WorkflowRuntimeBuilder().registerWorkflow(OrderProcessingWorkflow.class);
    builder.registerActivity(NotifyActivity.class);
    builder.registerActivity(ProcessPaymentActivity.class);
    builder.registerActivity(RequestApprovalActivity.class);
    builder.registerActivity(ReserveInventoryActivity.class);
    builder.registerActivity(UpdateInventoryActivity.class);

    // Build and then start the workflow runtime pulling and executing tasks
    try (WorkflowRuntime runtime = builder.build()) {
      System.out.println("Start workflow runtime");
      runtime.start(false);
    }
```

## 代码实现

### WorkflowRuntimeBuilder

这个类在 dapr java sdk。

WorkflowRuntimeBuilder 的实现中，自己会保存 workflows 和 activities 信息，也会构建一个来自 DurableTask java sdk 的 DurableTaskGrpcWorkerBuilder 的实例。

```java
import com.microsoft.durabletask.DurableTaskGrpcWorkerBuilder;

public class WorkflowRuntimeBuilder {
  private static volatile WorkflowRuntime instance;
  private DurableTaskGrpcWorkerBuilder builder;
  private Logger logger;
  private Set<String> workflows = new HashSet<String>();
  private Set<String> activities = new HashSet<String>();

  /**
   * Constructs the WorkflowRuntimeBuilder.
   */
  public WorkflowRuntimeBuilder() {
    this.builder = new DurableTaskGrpcWorkerBuilder().grpcChannel(
                          NetworkUtils.buildGrpcManagedChannel(WORKFLOW_INTERCEPTOR));
    this.logger = Logger.getLogger(WorkflowRuntimeBuilder.class.getName());
  }
```

registerWorkflow() 方法的实现，除了将请求代理给 DurableTaskGrpcWorkerBuilder 之外，还自己保存到 workflows 集合中：

```java
  public <T extends Workflow> WorkflowRuntimeBuilder registerWorkflow(Class<T> clazz) {
    this.builder = this.builder.addOrchestration(
        new OrchestratorWrapper<>(clazz)
    );
    this.logger.log(Level.INFO, "Registered Workflow: " +  clazz.getSimpleName());
    this.workflows.add(clazz.getSimpleName());
    return this;
  }
```

registerActivity() 方法的实现类似，除了将请求代理给 DurableTaskGrpcWorkerBuilder 之外，还自己保存到 activities 集合中：

```java
  public <T extends WorkflowActivity> void registerActivity(Class<T> clazz) {
    this.builder = this.builder.addActivity(
        new ActivityWrapper<>(clazz)
    );
    this.logger.log(Level.INFO, "Registered Activity: " +  clazz.getSimpleName());
    this.activities.add(clazz.getSimpleName());
  }
```

OrchestratorWrapper 和 ActivityWrapper 负责将 class 包装为 TaskOrchestrationFactory 和 TaskActivityFactory。

build() 方法调用 DurableTaskGrpcWorkerBuilder 的 build() 方法构建出一个 DurableTaskGrpcWorker ，然后传递给 WorkflowRuntime 的新实例。

```java
  public WorkflowRuntime build() {
    if (instance == null) {
      synchronized (WorkflowRuntime.class) {
        if (instance == null) {
          instance = new WorkflowRuntime(this.builder.build());
        }
      }
    }
    this.logger.log(Level.INFO, "Successfully built dapr workflow runtime");
    return instance;
  }
```

### DurableTaskGrpcWorkerBuilder

这个类在durabletask java sdk中。

DurableTaskGrpcWorkerBuilder 保存 orchestrationFactories 和 activityFactories，还有和 sidecar 连接的一些信息如端口，grpc channel：

```java
public final class DurableTaskGrpcWorkerBuilder {
    final HashMap<String, TaskOrchestrationFactory> orchestrationFactories = new HashMap<>();
    final HashMap<String, TaskActivityFactory> activityFactories = new HashMap<>();
    int port;
    Channel channel;
    DataConverter dataConverter;
    Duration maximumTimerInterval;
......
}
```

addOrchestration() 将 TaskOrchestrationFactory 保存到 orchestrationFactories 中，key为 name：

```java
    public DurableTaskGrpcWorkerBuilder addOrchestration(TaskOrchestrationFactory factory) {
        String key = factory.getName();
        ......
        this.orchestrationFactories.put(key, factory);
        return this;
    }
```

类似的, addActivity() 将 TaskActivityFactory 保存到 activityFactories 中，key为 name：

```java
    public DurableTaskGrpcWorkerBuilder addActivity(TaskActivityFactory factory) {
        String key = factory.getName();
        ......
        this.activityFactories.put(key, factory);
        return this;
    }
```

build() 方法构建出 DurableTaskGrpcWorker() 对象：

```java
    public DurableTaskGrpcWorker build() {
        return new DurableTaskGrpcWorker(this);
    }
```

DurableTaskGrpcWorker 的构造函数中会保存注册好的 orchestrationFactories 和 activityFactories，然后构建 TaskHubSidecarServiceGrpc 对象作为 sidecarClient，用于后续和 dapr sidecar 交互：

```java
public final class DurableTaskGrpcWorker implements AutoCloseable {
    private final HashMap<String, TaskOrchestrationFactory> orchestrationFactories = new HashMap<>();
    private final HashMap<String, TaskActivityFactory> activityFactories = new HashMap<>();

    private final TaskHubSidecarServiceBlockingStub sidecarClient;

    DurableTaskGrpcWorker(DurableTaskGrpcWorkerBuilder builder) {
        this.orchestrationFactories.putAll(builder.orchestrationFactories);
        this.activityFactories.putAll(builder.activityFactories);

        Channel sidecarGrpcChannel;
        if (builder.channel != null) {
            // The caller is responsible for managing the channel lifetime
            this.managedSidecarChannel = null;
            sidecarGrpcChannel = builder.channel;
        } else {
            // Construct our own channel using localhost + a port number
            int port = DEFAULT_PORT;
            if (builder.port > 0) {
                port = builder.port;
            }

            // Need to keep track of this channel so we can dispose it on close()
            this.managedSidecarChannel = ManagedChannelBuilder
                    .forAddress("localhost", port)
                    .usePlaintext()
                    .build();
            sidecarGrpcChannel = this.managedSidecarChannel;
        }

        this.sidecarClient = TaskHubSidecarServiceGrpc.newBlockingStub(sidecarGrpcChannel);
        this.dataConverter = builder.dataConverter != null ? builder.dataConverter : new JacksonDataConverter();
        this.maximumTimerInterval = builder.maximumTimerInterval != null ? builder.maximumTimerInterval : DEFAULT_MAXIMUM_TIMER_INTERVAL;
    }
```

## 结论

dapr java sdk 中的 WorkflowRuntimeBuilder 和 durabletask java sdk 中的 DurableTaskGrpcWorkerBuilder，都是用来保住构建最终要使用的 WorkflowRuntime 和 DurableTaskGrpcWorker。