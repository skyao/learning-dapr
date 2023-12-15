---
title: "python"
linkTitle: "python"
weight: 20
date: 2021-01-29
description: >
  运行 dapr workflow 的 python quickstart
---

### 运行 quickstart

执行：

```bash
cd workflows/python/sdk/order-processor
pip3 install -r requirements.txt
dapr run --app-id order-processor --resources-path ../../../components/ -- python3 app.py
```

输出为：

```bash
ℹ️  Starting Dapr with id order-processor. HTTP Port: 32971. gRPC Port: 43899
ℹ️  Checking if Dapr sidecar is listening on HTTP port 32971
INFO[0000] starting Dapr Runtime -- version 1.11.3 -- commit 9f99c6adca78dfc04b8063974f27b3a7534ae798  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] log level set to: info                        app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] metrics server started on :45495/             app_id=order-processor instance=dapr15 scope=dapr.metrics type=log ver=1.11.3
INFO[0000] Resiliency configuration loaded               app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] standalone mode configured                    app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] app id: order-processor                       app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] mTLS is disabled. Skipping certificate request and tls validation  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Dapr trace sampler initialized: DaprTraceSampler(P=1.000000)  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] local service entry announced: order-processor -> 192.168.99.15:46027  app_id=order-processor component="mdns (nameResolution/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0000] Initialized name resolution to mdns           app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading components…                           app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding components to be processed  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Using 'statestore-actors' as actor state store  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Component loaded: statestore-actors (state.redis/v1)  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding components processed          app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Loading endpoints                             app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] Waiting for all outstanding http endpoints to be processed  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] All outstanding http endpoints processed      app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC proxy enabled                            app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :43899  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] Registering workflow engine for gRPC endpoint: [::]:43899  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.api type=log ver=1.11.3
INFO[0000] API gRPC server is running on port 43899      app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] enabled metrics http middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] enabled tracing http middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] HTTP server listening on TCP address: :32971  app_id=order-processor instance=dapr15 scope=dapr.runtime.http type=log ver=1.11.3
INFO[0000] http server is running on port 32971          app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] The request body size parameter is: 4         app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] gRPC server listening on TCP address: :46027  app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC tracing middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] Enabled gRPC metrics middleware               app_id=order-processor instance=dapr15 scope=dapr.runtime.grpc.internal type=log ver=1.11.3
INFO[0000] internal gRPC server is running on port 46027  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0000] App channel is not initialized. Did you configure an app-port?  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s  app_id=order-processor instance=dapr15 scope=dapr.runtime.actor type=log ver=1.11.3
INFO[0000] Configuring workflow engine with actors backend  app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0000] Registering component for dapr workflow engine...  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] initializing Dapr workflow component          app_id=order-processor component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
WARN[0000] failed to read from bindings: app channel not initialized   app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] dapr initialized. Status: Running. Init Elapsed 8ms  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0000] placement tables updated, version: 2          app_id=order-processor instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
ℹ️  Checking if Dapr sidecar is listening on GRPC port 43899
ℹ️  Dapr sidecar is up and running.
ℹ️  Updating metadata for appPID: 11366
ℹ️  Updating metadata for app command: python3 app.py
✅  You're up and running! Both Dapr and your app logs will appear here.

== APP == *** Welcome to the Dapr Workflow console app sample!
== APP == *** Using this app, you can place orders that start workflows.
== APP == 2023-09-27 07:44:02.567 durabletask-worker INFO: Starting gRPC worker that connects to 127.0.0.1:43899
INFO[0006] work item stream established by user-agent: [grpc-python/1.58.0 grpc-c/35.0.0 (linux; chttp2)]  app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
INFO[0006] worker started with backend dapr.actors/v1-alpha  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == 2023-09-27 07:44:02.579 durabletask-worker INFO: Successfully connected to 127.0.0.1:43899. Waiting for work items...
INFO[0006] Workflow engine started                       app_id=order-processor instance=dapr15 scope=dapr.runtime.wfengine type=log ver=1.11.3
== APP == item: InventoryItem(item_name=Paperclip, per_item_cost=5, quantity=100)
== APP == item: InventoryItem(item_name=Cars, per_item_cost=15000, quantity=100)
== APP == item: InventoryItem(item_name=Computers, per_item_cost=500, quantity=100)
== APP == ==========Begin the purchase of item:==========
== APP == Starting order workflow, purchasing 10 of cars
INFO[0006] Error processing operation DaprBuiltInActorNotFoundRetries. Retrying in 1s…  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
INFO[0009] placement tables updated, version: 3          app_id=order-processor instance=dapr15 scope=dapr.runtime.actor.internal.placement type=log ver=1.11.3
INFO[0010] Recovered processing operation DaprBuiltInActorNotFoundRetries.  app_id=order-processor instance=dapr15 scope=dapr.runtime type=log ver=1.11.3
WARN[0010] Redis does not support transaction rollbacks and should not be used in production as an actor state store.  app_id=order-processor component="statestore-actors (state.redis/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0010] created new workflow instance with ID '662f09df-fcfb-4ee1-bda0-1327e3a7116f'  app_id=order-processor component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
INFO[0010] 662f09df-fcfb-4ee1-bda0-1327e3a7116f: starting new 'order_processing_workflow' instance with ID = '662f09df-fcfb-4ee1-bda0-1327e3a7116f'.  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == app.py:49: UserWarning: The Workflow API is an Alpha version and is subject to change.
== APP ==   start_resp = daprClient.start_workflow(workflow_component=workflow_component,
== APP == app.py:86: UserWarning: The Workflow API is an Alpha version and is subject to change.
== APP ==   state = daprClient.get_workflow(instance_id=_id, workflow_component=workflow_component)
== APP == 2023-09-27 07:44:06.592 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 1 task(s) and 0 event(s).
== APP == INFO:NotifyActivity:Received order 662f09df-fcfb-4ee1-bda0-1327e3a7116f for 10 cars at $150000 !
== APP == 2023-09-27 07:44:06.605 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 1 task(s) and 0 event(s).
== APP == INFO:VerifyInventoryActivity:Verifying inventory for order 662f09df-fcfb-4ee1-bda0-1327e3a7116f of 10 cars
== APP == INFO:VerifyInventoryActivity:There are 100 Cars available for purchase
== APP == 2023-09-27 07:44:06.617 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 1 task(s) and 0 event(s).
== APP == INFO:RequestApprovalActivity:Requesting approval for payment of 150000 USD for 10 cars
== APP == 2023-09-27 07:44:06.627 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 1 task(s) and 1 event(s).
INFO[0020] Raised event manager_approval on workflow instance '662f09df-fcfb-4ee1-bda0-1327e3a7116f'  app_id=order-processor component="dapr (workflow.dapr/v1)" instance=dapr15 scope=dapr.contrib type=log ver=1.11.3
== APP == app.py:95: UserWarning: The Workflow API is an Alpha version and is subject to change.
== APP ==   state = daprClient.get_workflow(instance_id=_id, workflow_component=workflow_component)
== APP == app.py:79: UserWarning: The Workflow API is an Alpha version and is subject to change.
== APP ==   daprClient.raise_workflow_event(instance_id=_id, workflow_component=workflow_component,
== APP == 2023-09-27 07:44:16.598 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f Event raised: manager_approval
== APP == 2023-09-27 07:44:16.598 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 2 task(s) and 0 event(s).
== APP == INFO:NotifyActivity:Payment for order 662f09df-fcfb-4ee1-bda0-1327e3a7116f has been approved!
== APP == 2023-09-27 07:44:16.608 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 2 task(s) and 0 event(s).
== APP == INFO:ProcessPaymentActivity:Processing payment: 662f09df-fcfb-4ee1-bda0-1327e3a7116f for 10 cars at 150000 USD
== APP == INFO:ProcessPaymentActivity:Payment for request ID 662f09df-fcfb-4ee1-bda0-1327e3a7116f processed successfully
== APP == 2023-09-27 07:44:16.618 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 2 task(s) and 0 event(s).
== APP == INFO:UpdateInventoryActivity:Checking inventory for order 662f09df-fcfb-4ee1-bda0-1327e3a7116f for 10 cars
== APP == INFO:UpdateInventoryActivity:There are now 90 cars left in stock
== APP == 2023-09-27 07:44:16.633 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Waiting for 2 task(s) and 0 event(s).
== APP == INFO:NotifyActivity:Order 662f09df-fcfb-4ee1-bda0-1327e3a7116f has completed!
== APP == 2023-09-27 07:44:16.643 durabletask-worker INFO: 662f09df-fcfb-4ee1-bda0-1327e3a7116f: Orchestration completed with status: COMPLETED
INFO[0020] 662f09df-fcfb-4ee1-bda0-1327e3a7116f: 'order_processing_workflow' completed with a COMPLETED status.  app_id=order-processor instance=dapr15 scope=wfengine.backend type=log ver=1.11.3
== APP == Workflow completed! Result: Completed
== APP == Purchase of item is  Completed
```

