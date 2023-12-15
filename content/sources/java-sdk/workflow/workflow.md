---
title: "workflow定义"
linkTitle: "workflow定义"
weight: 10
date: 2022-07-24
description: >
  workflow定义
---



## workflow 

Workflow 定义定义很简单：

```java
public abstract class Workflow {
  // 默认构造函数应该可以不用写的
  public Workflow(){
  }

  public abstract WorkflowStub create();

  public void run(WorkflowContext ctx) {
    this.create().run(ctx);
  }
}
```

create() 方法定义创建 WorkflowStub 的模板方法，然后在 run() 方法通过执行 create() 方法创建 WorkflowStub ，在执行 WorkflowStub 的 run() 方法。



## WorkflowStub

WorkflowStub 是一个单方法的接口定义，用于实现函数编程，标注有 `java.lang.@FunctionalInterface` 注解。

```java
@FunctionalInterface
public interface WorkflowStub {
  void run(WorkflowContext ctx);
}
```

@FunctionalInterface 的 javadoc 描述如下：

> 一种信息性注解类型，用于表明接口类型声明是 Java 语言规范所定义的函数接口。从概念上讲，一个函数接口只有一个抽象方法。由于默认方法有一个实现，所以它们不是抽象方法。如果一个接口声明了一个覆盖 java.lang.Object 公共方法之一的抽象方法，该方法也不计入接口的抽象方法数，因为接口的任何实现都将有一个来自 java.lang.Object 或其他地方的实现。
>
> 请注意，函数接口的实例可以通过 lambda 表达式、方法引用或构造器引用来创建。
>
> 如果一个类型被注释为该注释类型，编译器必须生成一条错误信息，除非：
>
> - 该类型是接口类型，而不是注解类型、枚举或类。
> - 注解的类型满足函数接口的要求。
>
> 然而，无论接口声明中是否有 FunctionalInterface 注解，编译器都会将任何符合函数接口定义的接口视为函数接口。

## WorkflowContext

出乎意外的是 WorkflowContext 的定义超级复杂，远远不是一个 上下文 那么简单。

### WorkflowContext的基本方法

WorkflowContext 接口上定义了大量的方法，其中部分基本方法

```java
public interface WorkflowContext {
  // 通过这个方法传递 logger 对象以供在后续执行时打印日志
  Logger getLogger();

  // 获取 workflow 的 name
  String getName();

  // 获取 workflow instance 的 id
  String getInstanceId();

  //获取当前协调时间（UTC）
  Instant getCurrentInstant();

  // 完成当前 wofklow，输出是完成的workflow的序列化输出
  void complete(Object output);
  ......
}
```

### waitForExternalEvent()方法

WorkflowContext 接口上定义了三个 waitForExternalEvent() 接口方法和一个默认实现：

```java
public interface WorkflowContext {
  ......
  <V> Task<V> waitForExternalEvent(String name, Duration timeout, Class<V> dataType) throws TaskCanceledException;

  <V> Task<Void> waitForExternalEvent(String name, Duration timeout) throws TaskCanceledException;

  <V> Task<Void> waitForExternalEvent(String name) throws TaskCanceledException;

  default <V> Task<V> waitForExternalEvent(String name, Class<V> dataType) {
    try {
      return this.waitForExternalEvent(name, null, dataType);
    } catch (TaskCanceledException e) {
      // This should never happen because of the max duration
      throw new RuntimeException("An unexpected exception was throw while waiting for an external event.", e);
    }
  }
  ......
}
```

waitForExternalEvent 的 javadoc 描述如下：

> 等待名为 name 的事件发生，并返回一个 Task，该任务在收到事件时完成，或在超时时取消。
>
> 如果当前协调器尚未等待名为 name 的事件，那么事件将保存在协调器实例状态中，并在调用此方法时立即派发。即使当前协调器在收到事件前取消了等待操作，事件保存也会发生。
>
> 协调器可以多次等待同一事件名，因此允许等待多个同名事件。协调器收到的每个外部事件将只完成本方法返回的一个任务。

特别注意：  这个 Task 的类型是 `com.microsoft.durabletask.Task` ，直接用在 dapr workflow 的接口定义上，意味着 dapr workflow 彻底和 durabletask 绑定。

### callActivity()方法

WorkflowContext 接口上定义了 callActivity() 接口方法和多个默认方法来重写不同参数的 callActivity() 方法

```java
public interface WorkflowContext {
  ......
  <V> Task<V> callActivity(String name, Object input, TaskOptions options, Class<V> returnType);

  default Task<Void> callActivity(String name) {
    return this.callActivity(name, null, null, Void.class);
  }

  default Task<Void> callActivity(String name, Object input) {
    return this.callActivity(name, input, null, Void.class);
  }

  default <V> Task<V> callActivity(String name, Class<V> returnType) {
    return this.callActivity(name, null, null, returnType);
  }

  default <V> Task<V> callActivity(String name, Object input, Class<V> returnType) {
    return this.callActivity(name, input, null, returnType);
  }

  default Task<Void> callActivity(String name, Object input, TaskOptions options) {
    return this.callActivity(name, input, options, Void.class);
  }
  ......
}
```

waitForExternalEvent 的 javadoc 描述如下：

> 使用指定的 input 异步调用一个 activity，并在 activity 完成时返回一个新的 task。如果 activity 成功完成，返回的 task 值将是 task 的输出。如果 activity 失败，返回的 task 将以 TaskFailedException 异常完成。



### isReplaying() 方法

isReplaying() 用来判断当前工作流当前是否正在重放之前的执行：

```java
public interface WorkflowContext {
  ......
  boolean isReplaying();
}
```

waitForExternalEvent 的 javadoc 描述如下：

> 获取一个值，指示工作流当前是否正在重放之前的执行。
>
> 工作流函数从内存中卸载后会进行 "重放"，以重建本地变量状态。在重放过程中，先前执行的任务将自动使用存储在工作流历史记录中的先前查看值完成。一旦工作流达到不再重放现有历史记录的程度，此方法将返回 false。
>
> 如果您的逻辑只需要在不重放时运行，则可以使用此方法。例如，某些类型的应用程序日志在作为重放的一部分进行复制时可能会变得过于嘈杂。应用程序代码可以检查函数是否正在重放，然后在该值为 false 时发出日志语句。



### allOf()和 anyOf()方法



```java
  <V> Task<List<V>> allOf(List<Task<V>> tasks) throws CompositeTaskFailedException;

  Task<Task<?>> anyOf(List<Task<?>> tasks);

  default Task<Task<?>> anyOf(Task<?>... tasks) {
    return this.anyOf(Arrays.asList(tasks));
  }
```

allOf 的 javadoc 描述如下：

> 返回一个新任务，该任务在所有给定任务完成后完成。如果任何给定任务在完成时出现异常，返回的任务也会在完成时出现 CompositeTaskFailedException，其中包含第一次遇到的故障的详细信息。返回的任务值是给定任务返回值的有序列表。如果没有提供任务，则返回值为空的已完成任务。
>
> 该方法适用于在继续协调的下一步之前等待一组独立任务的完成，如下面的示例：
>
> Task<String> t1 = ctx.callActivity("MyActivity", String.class)；
>  Task<String> t2 = ctx.callActivity("MyActivity", String.class)；
>  Task<String> t3 = ctx.callActivity("MyActivity", String.class)；
>
>  List<String> orderedResults = ctx.allOf(List.of(t1, t2, t3)).await()；
>
> 任何给定任务出现异常都会导致非受查的 CompositeTaskFailedException 异常。可以通过检查该异常来获取单个任务的失败详情。
>
> try {
>      List<String> orderedResults = ctx.allOf(List.of(t1, t2, t3)).await()；
>  } catch (CompositeTaskFailedException e) {
>      List exceptions = e.getExceptions()
>  }
>  }

特别注意：  这个 CompositeTaskFailedException 的类型是 `com.microsoft.durabletask.CompositeTaskFailedException` ，直接用在 dapr workflow 的接口定义上，意味着 dapr workflow 彻底和 durabletask 绑定。

anyOf 的 javadoc 描述如下：

> 当任何给定任务完成时，返回一个已完成的新任务。新任务的值是已完成任务对象的引用。如果没有提供任务，则返回一个永不完成的任务。
>
> 该方法适用于等待多个并发任务，并在第一个任务完成时执行特定于任务的操作，如下面的示例：
>
> Task<Void> event1 = ctx.waitForExternalEvent("Event1")；
>  Task<Void> event2 = ctx.waitForExternalEvent("Event2")；
>  Task<Void> event3 = ctx.waitForExternalEvent("Event3")；
>
>  Task<?> winner = ctx.anyOf(event1、event2、event3).await()；
>  如果（winner == event1）{
>      // ...
>  } else if (winner == event2) { // ...
>      // ...
>  } else if (winner == event3) { // ...
>      // ...
>  }
>
> anyOf 方法还可用于实现长时间超时，如下面的示例：
>
> Task<Void> activityTask = ctx.callActivity("SlowActivity")；
>  Task<Void> timeoutTask = ctx.createTimer(Duration.ofMinutes(30))；
>
>  Task<?> winner = ctx.anyOf(activityTask, timeoutTask).await()；
>  如果（winner == activityTask）{
>      // 完成情况
>  } else {
>      // 超时情况
>  }



### createTimer()方法

创建一个在指定延迟后过期的 durable timer。

指定较长的延迟（例如，几天或更长时间的延迟）可能会导致创建多个内部管理的 durable timer。协调器代码不需要意识到这种行为。不过，框架日志和存储的历史状态中可能会显示这种行为。

```java
  Task<Void> createTimer(Duration duration);

  default Task<Void> createTimer(ZonedDateTime zonedDateTime) {
    throw new UnsupportedOperationException("This method is not implemented.");
  }

```

### getInput()方法

getInput() 方法获取当前任务协调器的反序列化输入。

```java
<V> V getInput(Class<V> targetType);
```

### callSubWorkflow()

callSubWorkflow() 方法异步调用另一个工作流作为子工作流：

```java
  default Task<Void> callSubWorkflow(String name) {
    return this.callSubWorkflow(name, null);
  }

  default Task<Void> callSubWorkflow(String name, Object input) {
    return this.callSubWorkflow(name, input, null);
  }

  default <V> Task<V> callSubWorkflow(String name, Object input, Class<V> returnType) {
    return this.callSubWorkflow(name, input, null, returnType);
  }

  default <V> Task<V> callSubWorkflow(String name, Object input, String instanceID, Class<V> returnType) {
    return this.callSubWorkflow(name, input, instanceID, null, returnType);
  }

  default Task<Void> callSubWorkflow(String name, Object input, String instanceID, TaskOptions options) {
    return this.callSubWorkflow(name, input, instanceID, options, Void.class);
  }

  <V> Task<V> callSubWorkflow(String name,
                              @Nullable Object input,
                              @Nullable String instanceID,
                              @Nullable TaskOptions options,
                              Class<V> returnType);
```

callSubWorkflow() 的 javadoc 描述如下：

> 异步调用另一个工作流作为子工作流，并在子工作流完成时返回一个任务。如果子工作流成功完成，返回的任务值将是 activity 的输出。如果子工作流失败，返回的任务将以 TaskFailedException 异常完成。
>
> 子工作流有自己的 instance ID、历史和状态，与启动它的父工作流无关。将大型协调分解为子工作流有很多好处：
>
> - 将大型协调拆分成一系列较小的子工作流可以使代码更易于维护。
> - 如果协调逻辑需要协调大量任务，那么在多个计算节点上并发分布协调逻辑就非常有用。
> - 通过保持较小的父协调历史记录，可以减少内存使用和 CPU 开销。
>
> 缺点是启动子工作流和处理其输出会产生开销。这通常只适用于非常小的协调。
>
> 由于子工作流独立于父工作流，因此终止父协调不会影响任何子工作流。

### continueAsNew()

callSubWorkflow() 方法使用新输入重启协调并清除其历史记录：

```java
  default void continueAsNew(Object input) {
    this.continueAsNew(input, true);
  }

  void continueAsNew(Object input, boolean preserveUnprocessedEvents);
}
```

continueAsNew() 的 javadoc 描述如下：

> 使用新输入重启协调并清除其历史记录。
>
> 该方法主要针对永恒协调(eternal orchestrations)，即可能永远无法完成的协调。它的工作原理是重新启动协调，为其提供新的输入，并截断现有的协调历史。它允许协调无限期地继续运行，而不会让其历史记录无限制地增长。定期截断历史记录的好处包括降低内存使用率、减少存储容量，以及在重建状态时缩短协调器重播时间。
>
> 当协调器调用 continueAsNew 时，任何未完成任务的结果都将被丢弃。例如，如果计划了一个定时器，但在定时器启动前调用了 continueAsNew，那么定时器事件将被丢弃。唯一的例外是外部事件。默认情况下，如果协调收到外部事件但尚未处理，则会通过调用 waitForExternalEvent 将该事件保存在协调状态单元中。即使协调器使用 continueAsNew 重新启动，这些事件也会保留在内存中。可以通过为 preserveUnprocessedEvents 参数值指定 false 来禁用此行为。
>
> 协调器实现应在调用 continueAsNew 方法后立即完成。
