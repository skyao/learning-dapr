---
title: "运行Java App"
linkTitle: "App"
weight: 10
date: 2021-02-24
description: >
  以 debug 的方式运行 workflow Java App
---

以 `dapr/quickstarts` 仓库下的 `workflows/java/sdk/order-processor` 这个 java app 为例。




```bash
version: 1
common:
  resourcesPath: ../../components
apps:
  - appID: WorkflowConsoleApp
    appDirPath: ./order-processor/target
    command: ["java", "-jar", "OrderProcessingService-0.0.1-SNAPSHOT.jar", "io.dapr.quickstarts.workflows.WorkflowConsoleApp"]
```

## 参考

- https://code.visualstudio.com/docs/java/java-debugging