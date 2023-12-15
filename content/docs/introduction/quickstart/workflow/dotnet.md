---
title: ".net"
linkTitle: ".net"
weight: 10
date: 2021-01-29
description: >
  运行 dapr workflow 的 .net quickstart
---

### 运行 quickstart

执行：

```bash
cd workflows/csharp/sdk/order-processor
dapr run --app-id order-processor dotnet run
```

输出为：

```bash
ℹ️  Starting Dapr with id order-processor. HTTP Port: 43787. gRPC Port: 36499
ℹ️  Checking if Dapr sidecar is listening on HTTP port 43787
INFO[0000] starting Dapr Runtime -- version 1.11.3 -- commit 9f99c6adca78dfc04b8063974f27b3a7534ae798  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] log level set to: info                        app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] metrics server started on :45063/             app_id=order-processor instance=dapr15 scope=dapr.metrics type=log ver=1.11.3
INFO[0000] Resiliency configuration loaded               app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] standalone mode configured                    app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] app id: order-processor                       app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] mTLS is disabled. Skipping certificate request and tls validation  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Dapr trace sampler initialized: DaprTraceSampler(P=1.000000)  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] local service entry announced: order-processor -> 192.168.99.15:46019  app_id=order-processor component="mdns (nameResolution/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0000] Initialized name resolution to mdns           app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading components…                           app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Component loaded: pubsub (pubsub.redis/v1)    app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding components to be processed  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Using 'statestore' as actor state store       app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Component loaded: statestore (state.redis/v1)  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding components processed          app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading endpoints                             app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding http endpoints to be processed  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding http endpoints processed      app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC proxy enabled                            app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :36499  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Registering workflow engine for gRPC endpoint: [::]:36499  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] API gRPC server is running on port 36499      app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] enabled metrics http middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] enabled tracing http middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] HTTP server listening on TCP address: :43787  app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] http server is running on port 43787          app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] The request body size parameter is: 4         app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :46019  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] internal gRPC server is running on port 46019  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0000] App channel is not initialized. Did you configure an app-port?  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s  app_id=order-processor instance=dapr15 scope=dapr.runtime.actor type=log ver=1.11.3
INFO[0000] Configuring workflow engine with actors backend  app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0000] Registering component for dapr workflow engine...  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] initializing Dapr workflow component          app_id=order-processor component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
WARN[0000] app channel not initialized, make sure -app-port is specified if pubsub subscription is required  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0000] failed to read from bindings: app channel not initialized   app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] dapr initialized. Status: Running. Init Elapsed 10ms  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] placement tables updated, version: 0          app_id=order-processor instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
ℹ️  Checking if Dapr sidecar is listening on GRPC port 36499
ℹ️  Dapr sidecar is up and running.
ℹ️  Updating metadata for appPID: 5624
ℹ️  Updating metadata for app command: dotnet run
✅  You're up and running! Both Dapr and your app logs will appear here.

== APP == info: Microsoft.DurableTask[1]
== APP ==       Durable Task worker is connecting to sidecar at localhost:36499.
== APP == info: Microsoft.Hosting.Lifetime[0]
== APP ==       Application started. Press Ctrl+C to shut down.
== APP == info: Microsoft.Hosting.Lifetime[0]
== APP ==       Hosting environment: Production
== APP == info: Microsoft.Hosting.Lifetime[0]
== APP ==       Content root path: /home/sky/work/code/dapr/quickstarts/workflows/csharp/sdk/order-processor
== APP == Starting workflow 375d349f purchasing 10 Cars
INFO[0003] Error processing operation DaprBuiltInActorNotFoundRetries. Retrying in 1s…  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
== APP == info: Microsoft.DurableTask[4]
== APP ==       Sidecar work-item streaming connection established.
INFO[0003] work item stream established by user-agent: [grpc-dotnet/2.50.0 (.NET 6.0.22; CLR 6.0.22; net6.0; linux; x64)]  app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0003] worker started with backend dapr.actors/v1-alpha  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
INFO[0003] Workflow engine started                       app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0006] placement tables updated, version: 1          app_id=order-processor instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
INFO[0006] Recovered processing operation DaprBuiltInActorNotFoundRetries.  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0006] Redis does not support transaction rollbacks and should not be used in production as an actor state store.  app_id=order-processor component="statestore (state.redis/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0006] created new workflow instance with ID '375d349f'  app_id=order-processor component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0006] 375d349f: starting new 'OrderProcessingWorkflow' instance with ID = '375d349f'.  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == info: WorkflowConsoleApp.Activities.NotifyActivity[0]
== APP ==       Received order 375d349f for 10 Cars at $15000
== APP == Your workflow has started. Here is the status of the workflow: Running
== APP == info: WorkflowConsoleApp.Activities.ReserveInventoryActivity[0]
== APP ==       Reserving inventory for order 375d349f of 10 Cars
== APP == info: WorkflowConsoleApp.Activities.ReserveInventoryActivity[0]
== APP ==       There are: 100, Cars available for purchase
== APP == info: WorkflowConsoleApp.Activities.ProcessPaymentActivity[0]
== APP ==       Processing payment: 375d349f for 10 Cars at $15000
== APP == info: WorkflowConsoleApp.Activities.ProcessPaymentActivity[0]
== APP ==       Payment for request ID '375d349f' processed successfully
== APP == info: WorkflowConsoleApp.Activities.UpdateInventoryActivity[0]
== APP ==       Checking Inventory for: Order# 375d349f for 10 Cars
== APP == info: WorkflowConsoleApp.Activities.UpdateInventoryActivity[0]
== APP ==       There are now: 90 Cars left in stock
== APP == info: WorkflowConsoleApp.Activities.NotifyActivity[0]
== APP ==       Order 375d349f has completed!
INFO[0021] 375d349f: 'OrderProcessingWorkflow' completed with a COMPLETED status.  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == Workflow Status: Completed
INFO[0021] work item stream closed                       app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
✅  Exited App successfully
ℹ️  
terminated signal received: shutting down
✅  Exited Dapr successfully

```


### 存在问题

.net quickstart 运行没问题，但是 zipkin 显示的 trace 和文档中不一致，startinstance 这个 span 没有了，导致无法看到调用关系。

等待修复。