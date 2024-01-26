---
title: "流程概述"
linkTitle: "概述"
weight: 10
date: 2021-02-24
description: >
  workflow app start流程概述
---

## 流程整体

workflow app 启动时，典型代码如下：

```java
    // Register the OrderProcessingWorkflow and its activities with the builder.
    WorkflowRuntimeBuilder builder = new WorkflowRuntimeBuilder().registerWorkflow(OrderProcessingWorkflow.class);
    builder.registerActivity(NotifyActivity.class);
    builder.registerActivity(ProcessPaymentActivity.class);
    builder.registerActivity(RequestApprovalActivity.class);
    builder.registerActivity(ReserveInventoryActivity.class);
    builder.registerActivity(UpdateInventoryActivity.class);

    // Build and then start the workflow runtime pulling and executing tasks
    try (WorkflowRuntime runtime = builder.build()) {
      System.out.println("Start workflow runtime");
      runtime.start(false);
    }
```

这个过程中，注册了 workflow 和 activity，然后 start workflow runtime。workflow runtime 会启动 worker，从 dapr sidecar 持续获取工作任务，包括 workflow task 和 activity task，然后执行这些任务并把任务结果返回给到 dapr sidecar。

```plantuml
@startuml
participant "Workflow App" as WorkflowApp
participant "Dapr Sidecar" as DaprSidecar

WorkflowApp -> WorkflowApp: registerWorkflow()

WorkflowApp -> WorkflowApp: registerActivity()

WorkflowApp -[#red]> WorkflowApp: WorkflowRuntime.start()


WorkflowApp -> DaprSidecar: WorkflowRuntime.getWorkItems()
DaprSidecar --> WorkflowApp: 

loop has next task

alt is orchestration task

WorkflowApp -> WorkflowApp: execute orchestration task
WorkflowApp -> DaprSidecar: completeOrchestratorTask()
DaprSidecar --> WorkflowApp: 

else is activity task

WorkflowApp -> WorkflowApp: execute activity task
WorkflowApp -> DaprSidecar: completeActivityTask()
DaprSidecar --> WorkflowApp: 

end

end

@enduml
```

## 详细流程

### register workflow 

```plantuml
@startuml
participant "Workflow App" as WorkflowApp
participant "Dapr Java SDK" as DaprJavaSDK
participant "DurableTask Java SDK" as DurableTaskJavaSDK

WorkflowApp -> DaprJavaSDK: registerWorkflow()
DaprJavaSDK -> DurableTaskJavaSDK: addOrchestration()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 

@enduml
```


### register activity 

```plantuml
@startuml
participant "Workflow App" as WorkflowApp
participant "Dapr Java SDK" as DaprJavaSDK
participant "DurableTask Java SDK" as DurableTaskJavaSDK

WorkflowApp -> DaprJavaSDK: registerActivity()
DaprJavaSDK -> DurableTaskJavaSDK: registerActivity()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 

@enduml
```

### start workflow runtime 

```plantuml
@startuml
participant "Workflow App" as WorkflowApp
participant "Dapr Java SDK" as DaprJavaSDK
participant "DurableTask Java SDK" as DurableTaskJavaSDK

WorkflowApp -> DaprJavaSDK: WorkflowRuntime.start()
DaprJavaSDK -> DurableTaskJavaSDK: worker.start()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 
@enduml
```

### worker execute tasks 

```plantuml
@startuml
participant "Workflow App" as WorkflowApp
participant "Dapr Java SDK" as DaprJavaSDK
participant "DurableTask Java SDK" as DurableTaskJavaSDK

WorkflowApp -> DaprJavaSDK: registerWorkflow()
DaprJavaSDK -> DurableTaskJavaSDK: addOrchestration()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 

WorkflowApp -> DaprJavaSDK: registerActivity()
DaprJavaSDK -> DurableTaskJavaSDK: registerActivity()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 

WorkflowApp -> DaprJavaSDK: WorkflowRuntime.start()
DaprJavaSDK -> DurableTaskJavaSDK: worker.start()
DurableTaskJavaSDK --> DaprJavaSDK
DaprJavaSDK --> WorkflowApp: 
@enduml
```