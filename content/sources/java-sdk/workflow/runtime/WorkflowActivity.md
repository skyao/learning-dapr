---
title: "WorkflowActivity实现"
linkTitle: "WorkflowActivity"
weight: 60
date: 2022-07-24
description: >
  WorkflowActivity的代码实现
---

## WorkflowActivity接口定义

WorkflowActivity接口定义了 Activity

```java
public interface WorkflowActivity {
  /**
   * 执行活动逻辑并返回一个值，该值将被序列化并返回给调用的协调器。
   *
   * @param ctx 提供有关当前活动执行的信息，如活动名称和协调程序提供给它的输入数据。
   * @return 要返回给调用协调器的任何可序列化值。
   */
  Object run(WorkflowActivityContext ctx);
}
```

WorkflowActivity 的 javadoc 描述如下：

> 任务活动实现的通用接口。
>
> 活动(Activity)是 durable task 协调的基本工作单位。活动(Activity)是在业务流程中进行协调的任务。例如，您可以创建一个协调器来处理订单。这些任务包括检查库存、向客户收费和创建装运。每个任务都是一个单独的活动(Activity)。这些活动(Activity)可以串行执行、并行执行或两者结合执行。
>
> 与任务协调器不同的是，活动(Activity)在工作类型上不受限制。活动(Activity)函数经常用于进行网络调用或运行 CPU 密集型操作。活动(Activity)还可以将数据返回给协调器函数。 durable task 运行时保证每个被调用的活动(Activity)函数在协调执行期间至少被执行一次。
>
> 由于活动(Activity)只能保证至少执行一次，因此建议尽可能将活动(Activity)逻辑作为幂等逻辑来实现。
>
> 协调器使用 io.dapr.workflows.WorkflowContext.callActivity 方法重载之一来调度活动。

## WorkflowActivityContext

WorkflowActivityContext 简单包装了  durabletask 的 TaskActivityContext ：

```java
import com.microsoft.durabletask.TaskActivityContext;

public class WorkflowActivityContext implements TaskActivityContext {
  private final TaskActivityContext innerContext;

  public WorkflowActivityContext(TaskActivityContext context) throws IllegalArgumentException {
    if (context == null) {
      throw new IllegalArgumentException("Context cannot be null");
    }
    this.innerContext = context;
  }
  ......
}
```

TaskActivityContext 接口要求的 getName() 和 getInput() 方法都简单代理给了内部的 durabletask 的 TaskActivityContext ：

```java
  public String getName() {
    return this.innerContext.getName();
  }

  public <T> T getInput(Class<T> targetType) {
    return this.innerContext.getInput(targetType);
  }
```

> 备注：这样的包装并没有起到隔离 dapr sdk 和 durabletask sdk 的目的，还是紧密的耦合在一起，包装的意义何在？
