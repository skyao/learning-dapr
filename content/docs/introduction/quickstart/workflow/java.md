---
title: "Java"
linkTitle: "Java"
weight: 30
date: 2021-01-29
description: >
  运行 dapr workflow 的 Java quickstart
---

## 背景

Java 的 quickstart 还没有 merge：

https://github.com/dapr/quickstarts/pull/925

要执行的话需要用到 

https://github.com/skyao/quickstarts/tree/java-workflow-quickstart

### 运行 quickstart

执行：

```bash
cd workflows/java/sdk/order-processor
mvn clean install
dapr run --app-id WorkflowConsoleApp --resources-path ../../../components/ --dapr-grpc-port 50001 -- java -jar target/OrderProcessingService-0.0.1-SNAPSHOT.jar io.dapr.quickstarts.workflows.WorkflowConsoleApp
```

输出为：

```bash
ℹ️  Starting Dapr with id WorkflowConsoleApp. HTTP Port: 45063. gRPC Port: 50001
ℹ️  Checking if Dapr sidecar is listening on HTTP port 45063
INFO[0000] starting Dapr Runtime -- version 1.11.3 -- commit 9f99c6adca78dfc04b8063974f27b3a7534ae798  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] log level set to: info                        app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] metrics server started on :45793/             app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.metrics type=log ver=1.11.3
INFO[0000] Resiliency configuration loaded               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] standalone mode configured                    app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] app id: WorkflowConsoleApp                    app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] mTLS is disabled. Skipping certificate request and tls validation  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Dapr trace sampler initialized: DaprTraceSampler(P=1.000000)  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] local service entry announced: WorkflowConsoleApp -> 192.168.99.15:43665  app_id=WorkflowConsoleApp component="mdns (nameResolution/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0000] Initialized name resolution to mdns           app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading components…                           app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding components to be processed  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Using 'statestore-actors' as actor state store  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Component loaded: statestore-actors (state.redis/v1)  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding components processed          app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading endpoints                             app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding http endpoints to be processed  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding http endpoints processed      app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC proxy enabled                            app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :50001  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Registering workflow engine for gRPC endpoint: [::]:50001  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] API gRPC server is running on port 50001      app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] enabled metrics http middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] enabled tracing http middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] HTTP server listening on TCP address: :45063  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] http server is running on port 45063          app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] The request body size parameter is: 4         app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :43665  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] internal gRPC server is running on port 43665  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0000] App channel is not initialized. Did you configure an app-port?  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.actor type=log ver=1.11.3
INFO[0000] Configuring workflow engine with actors backend  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0000] Registering component for dapr workflow engine...  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] initializing Dapr workflow component          app_id=WorkflowConsoleApp component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
WARN[0000] failed to read from bindings: app channel not initialized   app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] dapr initialized. Status: Running. Init Elapsed 7ms  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] placement tables updated, version: 5          app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
ℹ️  Checking if Dapr sidecar is listening on GRPC port 50001
ℹ️  Dapr sidecar is up and running.
ℹ️  Updating metadata for appPID: 13886
ℹ️  Updating metadata for app command: java -jar target/OrderProcessingService-0.0.1-SNAPSHOT.jar io.dapr.quickstarts.workflows.WorkflowConsoleApp
✅  You're up and running! Both Dapr and your app logs will appear here.

== APP == *** Welcome to the Dapr Workflow console app sample!
== APP == *** Using this app, you can place orders that start workflows.
INFO[0003] placement tables updated, version: 6          app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
== APP == Start workflow runtime
== APP == Sep 27, 2023 7:51:22 AM com.microsoft.durabletask.DurableTaskGrpcWorker startAndBlock
== APP == INFO: Durable Task worker is connecting to sidecar at 127.0.0.1:50001.
INFO[0007] work item stream established by user-agent: [dapr-sdk-java/v1.10.0-SNAPSHOT grpc-java-netty/1.46.0]  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0007] worker started with backend dapr.actors/v1-alpha  app_id=WorkflowConsoleApp instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
INFO[0007] Workflow engine started                       app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
== APP == ==========Begin the purchase of item:==========
== APP == Starting order workflow, purchasing 10 of cars
INFO[0007] Error processing operation DaprBuiltInActorNotFoundRetries. Retrying in 1s…  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0010] placement tables updated, version: 7          app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
INFO[0010] Recovered processing operation DaprBuiltInActorNotFoundRetries.  app_id=WorkflowConsoleApp instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0010] Redis does not support transaction rollbacks and should not be used in production as an actor state store.  app_id=WorkflowConsoleApp component="statestore-actors (state.redis/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0010] d4d383b9-d615-489f-bfad-f94fad4c169c: starting new 'io.dapr.quickstarts.workflows.OrderProcessingWorkflow' instance with ID = 'd4d383b9-d615-489f-bfad-f94fad4c169c'.  app_id=WorkflowConsoleApp instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == scheduled new workflow instance of OrderProcessingWorkflow with instance ID: d4d383b9-d615-489f-bfad-f94fad4c169c
== APP == [Thread-0] INFO io.dapr.workflows.WorkflowContext - Starting Workflow: io.dapr.quickstarts.workflows.OrderProcessingWorkflow
== APP == [Thread-0] INFO io.dapr.workflows.WorkflowContext - Instance ID(order ID): d4d383b9-d615-489f-bfad-f94fad4c169c
== APP == [Thread-0] INFO io.dapr.workflows.WorkflowContext - Current Orchestration Time: 2023-09-27T07:51:26.473Z
== APP == [Thread-0] INFO io.dapr.workflows.WorkflowContext - Received Order: OrderPayload [itemName=cars, totalCost=150000, quantity=10]
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.NotifyActivity - Received Order: OrderPayload [itemName=cars, totalCost=150000, quantity=10]
== APP == workflow instance d4d383b9-d615-489f-bfad-f94fad4c169c started
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.ReserveInventoryActivity - Reserving inventory for order 'd4d383b9-d615-489f-bfad-f94fad4c169c' of 10 cars
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.ReserveInventoryActivity - There are 100 cars available for purchase
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.ReserveInventoryActivity - Reserved inventory for order 'd4d383b9-d615-489f-bfad-f94fad4c169c' of 10 cars
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.RequestApprovalActivity - Requesting approval for order: OrderPayload [itemName=cars, totalCost=150000, quantity=10]
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.RequestApprovalActivity - Approved requesting approval for order: OrderPayload [itemName=cars, totalCost=150000, quantity=10]
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.ProcessPaymentActivity - Processing payment: d4d383b9-d615-489f-bfad-f94fad4c169c for 10 cars at $150000
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.ProcessPaymentActivity - Payment for request ID 'd4d383b9-d615-489f-bfad-f94fad4c169c' processed successfully
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.UpdateInventoryActivity - Updating inventory for order 'd4d383b9-d615-489f-bfad-f94fad4c169c' of 10 cars
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.UpdateInventoryActivity - Updated inventory for order 'd4d383b9-d615-489f-bfad-f94fad4c169c': there are now 90 cars left in stock
== APP == there are now 90 cars left in stock
== APP == [Thread-0] INFO io.dapr.quickstarts.workflows.activities.NotifyActivity - Order completed! : d4d383b9-d615-489f-bfad-f94fad4c169c
INFO[0021] d4d383b9-d615-489f-bfad-f94fad4c169c: 'io.dapr.quickstarts.workflows.OrderProcessingWorkflow' completed with a COMPLETED status.  app_id=WorkflowConsoleApp instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == workflow instance completed, out is: {"processed":true}

```

