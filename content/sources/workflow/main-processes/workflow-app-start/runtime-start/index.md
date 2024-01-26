---
title: "启动workflow runtime的源码"
linkTitle: "启动workflow runtime"
weight: 30
date: 2021-02-24
description: >
  workflow app start流程中启动workflow runtime的源码
---

## 调用代码

workflow app 中启动 WorkflowRuntime 的典型使用代码如下：

```java
    // Build and then start the workflow runtime pulling and executing tasks
    try (WorkflowRuntime runtime = builder.build()) {
      System.out.println("Start workflow runtime");
      //这里写死了 block=false,不会 block
      runtime.start(false);
    }
```

## 代码实现

### WorkflowRuntime

这个类在 dapr java sdk。

WorkflowRuntime 只是对 DurableTaskGrpcWorker 的一个简单包装：

```java
public class WorkflowRuntime implements AutoCloseable {

  private DurableTaskGrpcWorker worker;

  public WorkflowRuntime(DurableTaskGrpcWorker worker) {
    this.worker = worker;
  }
  ......

  public void start(boolean block) {
    if (block) {
      this.worker.startAndBlock();
    } else {
      this.worker.start();
    }
  }
}

```


### DurableTaskGrpcWorker

这个类在durabletask java sdk中。

真实的实现代码在 DurableTaskGrpcWorker 中。

```java

  public void start(boolean block) {
    if (block) {
      this.worker.startAndBlock();
    } else {
      // 1. block写死false了，所以只会进入到这里
      this.worker.start();
    }
  }

  public void start() {
    // 2. 启动线程来执行 startAndBlock，所以是不阻塞的
    new Thread(this::startAndBlock).start();
  }
```

#### startAndBlock()方法

这是最关键的代码。

这里不展开，看下一章 workflow runtime 的运行。


