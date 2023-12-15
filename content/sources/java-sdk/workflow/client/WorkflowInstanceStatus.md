---
title: "WorkflowFailureDetails代码实现"
linkTitle: "WorkflowFailureDetails"
weight: 30
date: 2022-07-24
description: >
  WorkflowFailureDetails 的代码实现
---



WorkflowFailureDetails 只是非常简单的包装了 durabletask 的 FailureDetails

```java
public class WorkflowFailureDetails {

  FailureDetails workflowFailureDetails;

  /**
   * Class constructor.
   * @param failureDetails failure Details
   */
  public WorkflowFailureDetails(FailureDetails failureDetails) {
    this.workflowFailureDetails = failureDetails;
  }
```

然后代理各种方法：

```java
  public String getErrorType() {
    return workflowFailureDetails.getErrorType();
  }

  public String getErrorMessage() {
    return workflowFailureDetails.getErrorMessage();
  }

  public String getStackTrace() {
    return workflowFailureDetails.getStackTrace();
  }
```

