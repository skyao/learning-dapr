---
title: "DaprWorkflowContextImpl实现"
linkTitle: "DaprWorkflowContextImpl"
weight: 20
date: 2022-07-24
description: >
  DaprWorkflowContextImpl实现
---



### 类定义

DaprWorkflowContextImpl 类实现了 WorkflowContext 接口，实现上采用代理给内部字段 innerContext，这是一个 `com.microsoft.durabletask.TaskOrchestrationContext`

```java
import com.microsoft.durabletask.TaskOrchestrationContext;

public class DaprWorkflowContextImpl implements WorkflowContext {
  private final TaskOrchestrationContext innerContext;
  private final Logger logger;
  ......
}
```

构造函数只是简单赋值，加了一些必要的 null 检测：

```java
public DaprWorkflowContextImpl(TaskOrchestrationContext context) throws IllegalArgumentException {
    this(context, LoggerFactory.getLogger(WorkflowContext.class));
  }

  public DaprWorkflowContextImpl(TaskOrchestrationContext context, Logger logger) throws IllegalArgumentException {
    if (context == null) {
      throw new IllegalArgumentException("Context cannot be null");
    }
    if (logger == null) {
      throw new IllegalArgumentException("Logger cannot be null");
    }

    this.innerContext = context;
    this.logger = logger;
  }
```

### 方法实现

除 getLogger() 外的所有方法的实现都是简单的代理给 innerContext 的同名方法：

```java
  public Logger getLogger() {
    if (this.innerContext.getIsReplaying()) {
      return NOPLogger.NOP_LOGGER;
    }
    return this.logger;
  }

  public String getName() {
    return this.innerContext.getName();
  }

  public String getInstanceId() {
    return this.innerContext.getInstanceId();
  }

  public Instant getCurrentInstant() {
    return this.innerContext.getCurrentInstant();
  }

  public boolean isReplaying() {
    return this.innerContext.getIsReplaying();
  }

  public <V> Task<V> callSubWorkflow(String name, @Nullable Object input, @Nullable String instanceID,
                                     @Nullable TaskOptions options, Class<V> returnType) {

    return this.innerContext.callSubOrchestrator(name, input, instanceID, options, returnType);
  }

  public void continueAsNew(Object input) {
    this.innerContext.continueAsNew(input);
  }
```

## 小结

这个类基本就是 `com.microsoft.durabletask.TaskOrchestrationContext` 的简单包裹，所有功能都代理给 `com.microsoft.durabletask.TaskOrchestrationContext`， 包括设计甚至方法名。

dapr 的 workflow 实现基本是完全绑定在 durabletask 上的。
