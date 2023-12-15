---
title: "WorkflowInstanceStatus代码实现"
linkTitle: "WorkflowInstanceStatus"
weight: 20
date: 2022-07-24
description: >
  WorkflowInstanceStatus 的代码实现
---



### 类定义和构造函数

WorkflowInstanceStatus 代表工作流实例当前状态的快照，包括元数据。

WorkflowInstanceStatus 的实现依然是包装 durabletask，内部是一个 durabletask 的 OrchestrationMetadata，以及 OrchestrationMetadata 携带的 FailureDetails：

```java
import com.microsoft.durabletask.FailureDetails;
import com.microsoft.durabletask.OrchestrationMetadata;

public class WorkflowInstanceStatus {

  private final OrchestrationMetadata orchestrationMetadata;

  @Nullable
  private final WorkflowFailureDetails failureDetails;
    
  public WorkflowInstanceStatus(OrchestrationMetadata orchestrationMetadata) {
    if (orchestrationMetadata == null) {
      throw new IllegalArgumentException("OrchestrationMetadata cannot be null");
    }
    this.orchestrationMetadata = orchestrationMetadata;
    FailureDetails details = orchestrationMetadata.getFailureDetails();
    if (details != null) {
      this.failureDetails = new WorkflowFailureDetails(details);
    } else {
      this.failureDetails = null;
    }
  }
```

获取 FailureDetails 之后将转为 dapr 的 WorkflowFailureDetails, 这里的细节在 WorkflowFailureDetails 类实现中展开。

### 各种代理方法

