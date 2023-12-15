---
title: "WorkflowRuntime实现"
linkTitle: "WorkflowRuntime"
weight: 10
date: 2022-07-24
description: >
  WorkflowRuntime的代码实现
---



WorkflowRuntime 简单封装了 durabletask 的 DurableTaskGrpcWorker：

```java
import com.microsoft.durabletask.DurableTaskGrpcWorker;

public class WorkflowRuntime implements AutoCloseable {

  private DurableTaskGrpcWorker worker;

  public WorkflowRuntime(DurableTaskGrpcWorker worker) {
    this.worker = worker;
  }
  ......   
}
```

然后将  start() 和 close() 方法简单的代理给 durabletask 的 DurableTaskGrpcWorker：

```java
  public void start() {
    this.start(true);
  }

  public void start(boolean block) {
    if (block) {
      this.worker.startAndBlock();
    } else {
      this.worker.start();
    }
  }

  public void close() {
    if (this.worker != null) {
      this.worker.close();
      this.worker = null;
    }
  }
```

