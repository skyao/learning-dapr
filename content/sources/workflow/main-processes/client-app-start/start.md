---
title: "client app start流程"
linkTitle: "start"
weight: 20
date: 2021-02-24
description: >
  client app start流程源码分析
---

## 客户端代码实现

### DaprWorkflowClient

Dapr java SDK 中的 DaprWorkflowClient，包裹了 durabletask java sdk 的 DurableTaskClient：

```java
public class DaprWorkflowClient implements AutoCloseable {

  private DurableTaskClient innerClient;
  private ManagedChannel grpcChannel;

  private DaprWorkflowClient(ManagedChannel grpcChannel) {
    this(createDurableTaskClient(grpcChannel), grpcChannel);
  }

  private DaprWorkflowClient(DurableTaskClient innerClient, ManagedChannel grpcChannel) {
    this.innerClient = innerClient;
    this.grpcChannel = grpcChannel;
  }

  private static DurableTaskClient createDurableTaskClient(ManagedChannel grpcChannel) {
    return new DurableTaskGrpcClientBuilder()
        .grpcChannel(grpcChannel)
        .build();
  }
  ......
}
```

scheduleNewWorkflow()方法代理给了 DurableTaskClient 的 scheduleNewOrchestrationInstance() 方法：

```java
  public <T extends Workflow> String scheduleNewWorkflow(Class<T> clazz, Object input, String instanceId) {
    return this.innerClient.scheduleNewOrchestrationInstance(clazz.getCanonicalName(), input, instanceId);
  }
  ```

### DurableTaskClient 和 DurableTaskGrpcClient

这两个类在 durabletask java sdk 中。

DurableTaskGrpcClient 的 scheduleNewOrchestrationInstance() 方法的实现：

```java
@Override
public String scheduleNewOrchestrationInstance(
        String orchestratorName,
        NewOrchestrationInstanceOptions options) {
    if (orchestratorName == null || orchestratorName.length() == 0) {
        throw new IllegalArgumentException("A non-empty orchestrator name must be specified.");
    }

    Helpers.throwIfArgumentNull(options, "options");

    CreateInstanceRequest.Builder builder = CreateInstanceRequest.newBuilder();
    builder.setName(orchestratorName);

    String instanceId = options.getInstanceId();
    if (instanceId == null) {
        instanceId = UUID.randomUUID().toString();
    }
    builder.setInstanceId(instanceId);

    String version = options.getVersion();
    if (version != null) {
        builder.setVersion(StringValue.of(version));
    }

    Object input = options.getInput();
    if (input != null) {
        String serializedInput = this.dataConverter.serialize(input);
        builder.setInput(StringValue.of(serializedInput));
    }

    Instant startTime = options.getStartTime();
    if (startTime != null) {
        Timestamp ts = DataConverter.getTimestampFromInstant(startTime);
        builder.setScheduledStartTimestamp(ts);
    }

    CreateInstanceRequest request = builder.build();
    CreateInstanceResponse response = this.sidecarClient.startInstance(request);
    return response.getInstanceId();
}
```

前面一大段都是为了构建 CreateInstanceRequest，然后最后调用 sidecarClient.startInstance() 方法去访问 sidecar 。

## proto 定义

TaskHubSidecarServiceBlockingStub 是根据 protobuf 文件生成的 grpc 代码，其 protobuf 定义在submodules/durabletask-protobuf/protos/orchestrator_service.proto 文件中。

```protobuf
service TaskHubSidecarService {
    ......
    // Starts a new orchestration instance.
    rpc StartInstance(CreateInstanceRequest) returns (CreateInstanceResponse);
    ......
}
```

CreateInstanceRequest 消息的定义为：

```protobuf
message CreateInstanceRequest {
    string instanceId = 1;
    string name = 2;
    google.protobuf.StringValue version = 3;
    google.protobuf.StringValue input = 4;
    google.protobuf.Timestamp scheduledStartTimestamp = 5;
    OrchestrationIdReusePolicy orchestrationIdReusePolicy = 6;
}
```

CreateInstanceResponse 信息的定义，很简单，只有一个 instanceId 字段。

```protobuf
message CreateInstanceResponse {
    string instanceId = 1;
}
```

## 服务器端代码实现

客户端和服务器端走的是 durable task 定义的 TaskHubSidecarService，这里特别要注意的是：不是 dapr API！

### StartInstance() 方法

StartInstance() 方法的服务器端代码实现在 durabletask-go 仓库的 `backend/executor.go` 文件中:

```go
func (g *grpcExecutor) StartInstance(ctx context.Context, req *protos.CreateInstanceRequest) (*protos.CreateInstanceResponse, error) {
	instanceID := req.InstanceId
	ctx, span := helpers.StartNewCreateOrchestrationSpan(ctx, req.Name, req.Version.GetValue(), instanceID)
	defer span.End()

	e := helpers.NewExecutionStartedEvent(req.Name, instanceID, req.Input, nil, helpers.TraceContextFromSpan(span))
	if err := g.backend.CreateOrchestrationInstance(ctx, e, WithOrchestrationIdReusePolicy(req.OrchestrationIdReusePolicy)); err != nil {
		return nil, err
	}

	return &protos.CreateInstanceResponse{InstanceId: instanceID}, nil
}
```

这里创建一个 ExecutionStartedEvent，然后调用 backend 的 CreateOrchestrationInstance() 方法。

### NewExecutionStartedEvent() 方法

这行代码的作用是构建一个带有 ExecutionStartedEvent 事件的 HistoryEvent：

```go
e := helpers.NewExecutionStartedEvent(req.Name, instanceID, req.Input, nil, helpers.TraceContextFromSpan(span))
```

具体实现为：

```go
func NewExecutionStartedEvent(
	name string,
	instanceId string,
	input *wrapperspb.StringValue,
	parent *protos.ParentInstanceInfo,
	parentTraceContext *protos.TraceContext,
) *protos.HistoryEvent {
	return &protos.HistoryEvent{
		EventId:   -1,
		Timestamp: timestamppb.New(time.Now()),
		EventType: &protos.HistoryEvent_ExecutionStarted{
			ExecutionStarted: &protos.ExecutionStartedEvent{
				Name:           name,
				ParentInstance: parent,
				Input:          input,
				OrchestrationInstance: &protos.OrchestrationInstance{
					InstanceId:  instanceId,
					ExecutionId: wrapperspb.String(uuid.New().String()),
				},
				ParentTraceContext: parentTraceContext,
			},
		},
	}
}
```

这个数据结构有点复杂，HistoryEvent 携带一个 ExecutionStartedEvent 类型（grpc 的 oneof）。

### CreateOrchestrationInstance() 方法

最关键的代码：

```go
if err := g.backend.CreateOrchestrationInstance(ctx, e, WithOrchestrationIdReusePolicy(req.OrchestrationIdReusePolicy)); err != nil {
  return nil, err
}
```

Backend 是一个 interface，CreateOrchestrationInstance() 方法定义如下：

```go
type Backend interface {
  // CreateOrchestrationInstance creates a new orchestration instance with a history event that
	// wraps a ExecutionStarted event.
	CreateOrchestrationInstance(context.Context, *HistoryEvent, ...OrchestrationIdReusePolicyOptions) error
  ......
}
```

这个 backend 可以有多种实现，durabletask-go 只负责进行调用，容许注入不同的实现。在 dapr workflow中，注入的是基于 dapr actor 的实现。

### daprd backend 的初始化

在 daprd sidecar 的代码实现中，这个 backend 是这样构建的，代码在 dapr/dapr 仓库的 `pkg/runtime/wfengine/wfengine.go` :

```go
func (wfe *WorkflowEngine) ConfigureGrpcExecutor() {
	// Enable lazy auto-starting the worker only when a workflow app connects to fetch work items.
	autoStartCallback := backend.WithOnGetWorkItemsConnectionCallback(func(ctx context.Context) error {
		// NOTE: We don't propagate the context here because that would cause the engine to shut
		//       down when the client disconnects and cancels the passed-in context. Once it starts
		//       up, we want to keep the engine running until the runtime shuts down.
		if err := wfe.Start(context.Background()); err != nil {
			// This can happen if the workflow app connects before the sidecar has finished initializing.
			// The client app is expected to continuously retry until successful.
			return fmt.Errorf("failed to auto-start the workflow engine: %w", err)
		}
		return nil
	})

	// Create a channel that can be used to disconnect the remote client during shutdown.
	wfe.disconnectChan = make(chan any, 1)
	disconnectHelper := backend.WithStreamShutdownChannel(wfe.disconnectChan)

	wfe.executor, wfe.registerGrpcServerFn = backend.NewGrpcExecutor(wfe.Backend, wfLogger, autoStartCallback, disconnectHelper)
}
```

 WorkflowEngine 的初始化代码在 `pkg/runtime/runtime.go` 中：

```go
	// Creating workflow engine after components are loaded
	wfe := wfengine.NewWorkflowEngine(a.runtimeConfig.id, a.globalConfig.GetWorkflowSpec(), a.processor.WorkflowBackend())
	wfe.ConfigureGrpcExecutor()
	a.workflowEngine = wfe
```

```go
	processor := processor.New(processor.Options{
		ID:             runtimeConfig.id,
		Namespace:      namespace,
		IsHTTP:         runtimeConfig.appConnectionConfig.Protocol.IsHTTP(),
		ActorsEnabled:  len(runtimeConfig.actorsService) > 0,
		Registry:       runtimeConfig.registry,
		ComponentStore: compStore,
		Meta:           meta,
		GlobalConfig:   globalConfig,
		Resiliency:     resiliencyProvider,
		Mode:           runtimeConfig.mode,
		PodName:        podName,
		Standalone:     runtimeConfig.standalone,
		OperatorClient: operatorClient,
		GRPC:           grpc,
		Channels:       channels,
	})
```

### ActorBackend 的实现

ActorBackend 实现了 durabletask-go 定义的 Backend 接口：

```go
type ActorBackend struct {
	orchestrationWorkItemChan chan *backend.OrchestrationWorkItem
	activityWorkItemChan      chan *backend.ActivityWorkItem
	startedOnce               sync.Once
	config                    actorsBackendConfig
	activityActorOpts         activityActorOpts
	workflowActorOpts         workflowActorOpts

	actorRuntime  actors.ActorRuntime
	actorsReady   atomic.Bool
	actorsReadyCh chan struct{}
}
```
CreateOrchestrationInstance() 方法的实现：

```go
func (abe *ActorBackend) CreateOrchestrationInstance(ctx context.Context, e *backend.HistoryEvent, opts ...backend.OrchestrationIdReusePolicyOptions) error {
	if err := abe.validateConfiguration(); err != nil {
		return err
	}

  // 对输入做必要的检查
	var workflowInstanceID string
	if es := e.GetExecutionStarted(); es == nil {
		return errors.New("the history event must be an ExecutionStartedEvent")
	} else if oi := es.GetOrchestrationInstance(); oi == nil {
		return errors.New("the ExecutionStartedEvent did not contain orchestration instance information")
	} else {
		workflowInstanceID = oi.GetInstanceId()
	}

	policy := &api.OrchestrationIdReusePolicy{}
	for _, opt := range opts {
		opt(policy)
	}

	eventData, err := backend.MarshalHistoryEvent(e)
	if err != nil {
		return err
	}

	requestBytes, err := json.Marshal(CreateWorkflowInstanceRequest{
		Policy:          policy,
		StartEventBytes: eventData,
	})
	if err != nil {
		return fmt.Errorf("failed to marshal CreateWorkflowInstanceRequest: %w", err)
	}

	// Invoke the well-known workflow actor directly, which will be created by this invocation request.
	// Note that this request goes directly to the actor runtime, bypassing the API layer.
	req := internalsv1pb.NewInternalInvokeRequest(CreateWorkflowInstanceMethod).
		WithActor(abe.config.workflowActorType, workflowInstanceID).
		WithData(requestBytes).
		WithContentType(invokev1.JSONContentType)
	start := time.Now()
	_, err = abe.actorRuntime.Call(ctx, req)
	elapsed := diag.ElapsedSince(start)
	if err != nil {
		// failed request to CREATE workflow, record count and latency metrics.
		diag.DefaultWorkflowMonitoring.WorkflowOperationEvent(ctx, diag.CreateWorkflow, diag.StatusFailed, elapsed)
		return err
	}
	// successful request to CREATE workflow, record count and latency metrics.
	diag.DefaultWorkflowMonitoring.WorkflowOperationEvent(ctx, diag.CreateWorkflow, diag.StatusSuccess, elapsed)
	return nil
}
```

关键代码在:

```go
_, err = abe.actorRuntime.Call(ctx, req)
```

这表明在 dapr 的 workflow backend 实现中，workflow 的调用是通过 dapr actor 来进行的。

然后注意这里的 req 的数据组装过程

```go
const	CreateWorkflowInstanceMethod = "CreateWorkflowInstance"

	requestBytes, err := json.Marshal(CreateWorkflowInstanceRequest{
		Policy:          policy,
		StartEventBytes: eventData,
	})

	req := internalsv1pb.NewInternalInvokeRequest(CreateWorkflowInstanceMethod).
		WithActor(abe.config.workflowActorType, workflowInstanceID).
		WithData(requestBytes).
		WithContentType(invokev1.JSONContentType)
```

actor id 被设置为当前 workflow 的 InstanceID，actor type 是配置的，具体的组装方式如下：

```go
		workflowActorType: actors.InternalActorTypePrefix + utils.GetNamespaceOrDefault(defaultNamespace) + utils.DotDelimiter + appID + utils.DotDelimiter + WorkflowNameLabelKey,
```

Method 被设置为 CreateWorkflowInstance，数据 requestBytes 是将整个 HistoryEvent 用 json 序列化之后的字节数组。

### ActorRuntime 的初始化

其中 ActorRuntime 是这样设置进来的：

```go
func (abe *ActorBackend) SetActorRuntime(ctx context.Context, actorRuntime actors.ActorRuntime) {
	abe.actorRuntime = actorRuntime
	if abe.actorsReady.CompareAndSwap(false, true) {
		close(abe.actorsReadyCh)
	}
}
```

调用的地方在 `pkg/runtime/runtime.go` 的 initWorkflowEngine() 方法中：

```go
func (a *DaprRuntime) initWorkflowEngine(ctx context.Context) error {
	wfComponentFactory := wfengine.BuiltinWorkflowFactory(a.workflowEngine)

	// If actors are not enabled, still invoke SetActorRuntime on the workflow engine with `nil` to unblock startup
	if abe, ok := a.workflowEngine.Backend.(interface {
		SetActorRuntime(ctx context.Context, actorRuntime actors.ActorRuntime)
	}); ok {
		log.Info("Configuring workflow engine with actors backend")
		var actorRuntime actors.ActorRuntime
		if a.runtimeConfig.ActorsEnabled() {
			actorRuntime = a.actor
		}
		abe.SetActorRuntime(ctx, actorRuntime)
	}
  ......
```

### actorRuntime的实现

ActorRuntime 的 interface 定义在文件 `pkg/actors/actors.go` 中：

```go
// ActorRuntime is the main runtime for the actors subsystem.
type ActorRuntime interface {
	Actors
	io.Closer
	Init(context.Context) error
	IsActorHosted(ctx context.Context, req *ActorHostedRequest) bool
	GetRuntimeStatus(ctx context.Context) *runtimev1pb.ActorRuntime
	RegisterInternalActor(ctx context.Context, actorType string, actor InternalActorFactory, actorIdleTimeout time.Duration) error
}
```

ActorRuntime 继承了 Actors interface，call()方法在这里定义：

```go
// Actors allow calling into virtual actors as well as actor state management.
type Actors interface {
	// Call an actor.
	Call(ctx context.Context, req *internalv1pb.InternalInvokeRequest) (*internalv1pb.InternalInvokeResponse, error)
  ......
}
```

实现代码也在同一个文件中，actorsRuntime struct：

```go
type actorsRuntime struct {
}	
```

actorsRuntime 的Call()方法的代码实现：

```go
func (a *actorsRuntime) Call(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	err := a.placement.WaitUntilReady(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to wait for placement readiness: %w", err)
	}

	actor := req.Actor()
	lar, err := a.placement.LookupActor(ctx, internal.LookupActorRequest{
		ActorType: actor.GetActorType(),
		ActorID:   actor.GetActorId(),
		Revision:  int(actor.GetRevision()),
	})
	if err != nil {
		return nil, err
	}
	var resp *invokev1.InvokeMethodResponse
	if a.isActorLocal(lar.Address, a.actorsConfig.Config.HostAddress, a.actorsConfig.Config.Port) {
		resp, err = a.callLocalActor(ctx, req)
	} else {
		resp, err = a.callRemoteActorWithRetry(ctx, retry.DefaultLinearRetryCount, retry.DefaultLinearBackoffInterval, a.callRemoteActor, lar.Address, lar.AppID, req)
	}

	if err != nil {
		if resp != nil && actorerrors.Is(err) {
			return resp, err
		}

		if resp != nil {
			resp.Close()
		}
		return nil, err
	}
	return resp, nil
}
```

关键代码在这里，调用 placement.LookupActor() 方法来查找要调用的目标actor的地址：

```go
	lar, err := a.placement.LookupActor(ctx, internal.LookupActorRequest{
		ActorType: actorType,
		ActorID:   actor.GetActorId(),
	})
```
