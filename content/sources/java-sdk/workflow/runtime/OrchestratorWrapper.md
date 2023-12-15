---
title: "OrchestratorWrapper实现"
linkTitle: "OrchestratorWrapper"
weight: 30
date: 2022-07-24
description: >
  OrchestratorWrapper的代码实现
---



### 背景

WorkflowRuntimeBuilder 的 registerWorkflow() 方法在注册 workflow 对象时，实际代理给 DurableTaskGrpcWorkerBuilder 的 addOrchestration() 方法：

```java
import com.microsoft.durabletask.TaskOrchestrationFactory;  

public <T extends Workflow> WorkflowRuntimeBuilder registerWorkflow(Class<T> clazz) {
    this.builder = this.builder.addOrchestration(
        new OrchestratorWrapper<>(clazz)
    );

    return this;
  }
```

而 addOrchestration() 方法的输入参数为 `com.microsoft.durabletask.TaskOrchestrationFactory`：

```java
public interface TaskOrchestrationFactory {
    String getName();
    TaskOrchestration create();
}
```

因此需要提供一个 TaskOrchestrationFactory 的实现。

### 类定义

OrchestratorWrapper 类实现了 `com.microsoft.durabletask.TaskOrchestrationFactory` 接口：

```java
class OrchestratorWrapper<T extends Workflow> implements TaskOrchestrationFactory {
  private final Constructor<T> workflowConstructor;
  private final String name;
  ......  
}
```

构造函数：

```java
  public OrchestratorWrapper(Class<T> clazz) {
    // 获取并设置 name
    this.name = clazz.getCanonicalName();
    try {
      // 获取 Constructor
      this.workflowConstructor = clazz.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
      throw new RuntimeException(
          String.format("No constructor found for workflow class '%s'.", this.name), e
      );
    }
  }
```

### 接口实现

TaskOrchestrationFactory 接口要求的 getName() 方法，直接返回前面获取的 name：

```java
  @Override
  public String getName() {
    return name;
  }
```

TaskOrchestrationFactory 接口要求的 create() 方法，要返回一个 durabletask 的 TaskOrchestration ，而 TaskOrchestration 是一个 @**FunctionalInterface**，仅有一个 run() 方法：

```java
@FunctionalInterface
public interface TaskOrchestration {
    void run(TaskOrchestrationContext ctx);
}
```

因此构建 TaskOrchestration 实例的方式被简写为：

```java
import com.microsoft.durabletask.TaskOrchestration;

  @Override
  public TaskOrchestration create() {
    return ctx -> {
      T workflow;
      try {
        // 通过 workflow 的构造器生成一个 workflow 实例
        workflow = this.workflowConstructor.newInstance();
      } catch (InstantiationException | IllegalAccessException | InvocationTargetException e) {
        throw new RuntimeException(
            String.format("Unable to instantiate instance of workflow class '%s'", this.name), e
        );
      }
      // 将 durable task 的 context 包装为 dapr 的 workflow context DaprWorkflowContextImpl
      // 然后执行 workflow.run()
      workflow.run(new DaprWorkflowContextImpl(ctx));
    };

  }
```

