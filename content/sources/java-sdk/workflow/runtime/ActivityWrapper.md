---
title: "ActivityWrapper实现"
linkTitle: "ActivityWrapper"
weight: 40
date: 2022-07-24
description: >
  ActivityWrapper的代码实现
---



### 背景

WorkflowRuntimeBuilder 的 registerActivity() 方法在注册 activity 对象时，实际代理给 DurableTaskGrpcWorkerBuilder 的 addActivity() 方法：

```java
import com.microsoft.durabletask.TaskOrchestrationFactory;  

  public <T extends WorkflowActivity> void registerActivity(Class<T> clazz) {
    this.builder = this.builder.addActivity(
        new ActivityWrapper<>(clazz)
    );
  }
```

而 addActivity() 方法的输入参数为 `com.microsoft.durabletask.TaskActivityFactory`：

```java
public interface TaskActivityFactory {
    String getName();
    TaskActivity create();
}
```

因此需要提供一个 TaskActivityFactory 的实现。

### 类定义

ActivityWrapper 类实现了 `com.microsoft.durabletask.TaskActivityFactory` 接口：

```java
public class ActivityWrapper<T extends WorkflowActivity> implements TaskActivityFactory {
  private final Constructor<T> activityConstructor;
  private final String name;
  ......  
}
```

构造函数：

```java
  public ActivityWrapper(Class<T> clazz) {
    this.name = clazz.getCanonicalName();
    try {
      this.activityConstructor = clazz.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
      throw new RuntimeException(
          String.format("No constructor found for activity class '%s'.", this.name), e
      );
    }
  }
```

### 接口实现

TaskActivityFactory 接口要求的 getName() 方法，直接返回前面获取的 name：

```java
  @Override
  public String getName() {
    return name;
  }
```

TaskActivityFactory 接口要求的 create() 方法，要返回一个 durabletask 的 TaskActivity ，而 TaskActivity 是一个 @**FunctionalInterface**，仅有一个 run() 方法：

```java
@FunctionalInterface
public interface TaskActivity {
    Object run(TaskActivityContext ctx);
}
```

因此构建 TaskActivity 实例的方式被简写为：

```java
import com.microsoft.durabletask.TaskActivity;

  @Override
  public TaskActivity create() {
    return ctx -> {
      Object result;
      T activity;
      
      try {
        activity = this.activityConstructor.newInstance();
      } catch (InstantiationException | IllegalAccessException | InvocationTargetException e) {
        throw new RuntimeException(
            String.format("Unable to instantiate instance of activity class '%s'", this.name), e
        );
      }

      result = activity.run(new WorkflowActivityContext(ctx));
      return result;
    };
  }
}
```

