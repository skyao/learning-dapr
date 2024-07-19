---
title: "actor实现流程"
linkTitle: "actor"
weight: 40
date: 2021-02-24
description: >
  client app start流程中actor实现的源码分析
---

## 背景

Call()方法的代码实现在 `pkg/actors/actors.go` 中：

```go
func (a *actorsRuntime) Call(ctx context.Context, req *internalv1pb.InternalInvokeRequest) (res *internalv1pb.InternalInvokeResponse, err error) {
	err = a.placement.WaitUntilReady(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to wait for placement readiness: %w", err)
	}

	// Do a lookup to check if the actor is local
	actor := req.GetActor()
	actorType := actor.GetActorType()
	lar, err := a.placement.LookupActor(ctx, internal.LookupActorRequest{
		ActorType: actorType,
		ActorID:   actor.GetActorId(),
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

	return res, nil
}
```

## callLocalActor()

调用本地，

```go
func (a *actorsRuntime) callLocalActor(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error) {
	actorTypeID := req.Actor()

	act := a.getOrCreateActor(actorTypeID)

	// Reentrancy to determine how we lock.
  ......

	// Replace method to actors method.
	msg := req.Message()
	originalMethod := msg.GetMethod()
	msg.Method = "actors/" + actorTypeID.GetActorType() + "/" + actorTypeID.GetActorId() + "/method/" + msg.GetMethod()

	// Reset the method so we can perform retries.
	defer func() {
		msg.Method = originalMethod
	}()

	// Original code overrides method with PUT. Why?
	if msg.GetHttpExtension() == nil {
		req.WithHTTPExtension(http.MethodPut, "")
	} else {
		msg.HttpExtension.Verb = commonv1pb.HTTPExtension_PUT //nolint:nosnakecase
	}

	appCh := a.getAppChannel(act.actorType)
	if appCh == nil {
		return nil, fmt.Errorf("app channel for actor type %s is nil", act.actorType)
	}

	policyDef := a.resiliency.ActorPostLockPolicy(act.actorType, act.actorID)

	// If the request can be retried, we need to enable replaying
	if policyDef != nil && policyDef.HasRetries() {
		req.WithReplay(true)
	}

	policyRunner := resiliency.NewRunnerWithOptions(ctx, policyDef,
		resiliency.RunnerOpts[*invokev1.InvokeMethodResponse]{
			Disposer: resiliency.DisposerCloser[*invokev1.InvokeMethodResponse],
		},
	)
	resp, err := policyRunner(func(ctx context.Context) (*invokev1.InvokeMethodResponse, error) {

		fmt.Printf("**** versioning ****: appCh.InvokeMethod(): msg.Method=%s\n", msg.Method)

		return appCh.InvokeMethod(ctx, req, "")
	})
	if err != nil {
		return nil, err
	}

	if resp == nil {
		return nil, errors.New("error from actor service: response object is nil")
	}

	if resp.Status().GetCode() != http.StatusOK {
		respData, _ := resp.RawDataFull()
		return nil, fmt.Errorf("error from actor service: %s", string(respData))
	}

	// The .NET SDK signifies Actor failure via a header instead of a bad response.
	if _, ok := resp.Headers()["X-Daprerrorresponseheader"]; ok {
		return resp, actorerrors.NewActorError(resp)
	}

	return resp, nil
}
```

最核心的调用时这一句，通过和 app 之间的通道，将请求发送到 app：

```go
appCh.InvokeMethod(ctx, req, "")
```

### appCh.InvokeMethod 的实现

`pkg/channel/grpc/grpc_channel.go` 

```go
	resp, err := g.appCallbackClient.OnInvoke(ctx, pd.GetMessage(), opts...)
```

### req 的组装过程

仔细看 req 的组装过程，req 是 `*invokev1.InvokeMethodRequest`

```go
type InvokeMethodRequest struct {
	replayableRequest

	r           *internalv1pb.InternalInvokeRequest
	dataObject  any
	dataTypeURL string
}
```

组装数据的代码在 `pkg/runtime/wfengine/backend.go`:

```go
CreateWorkflowInstanceMethod = "CreateWorkflowInstance"

func (be *actorBackend) CreateOrchestrationInstance(ctx context.Context, revision int, e *backend.HistoryEvent, opts ...backend.OrchestrationIdReusePolicyOptions) error {

	eventData, err := backend.MarshalHistoryEvent(e)

	requestBytes, err := json.Marshal(CreateWorkflowInstanceRequest{
		Policy:          policy,
		StartEventBytes: eventData,
	})

	req := invokev1.
		NewInvokeMethodRequest(CreateWorkflowInstanceMethod).
		WithActorRevision(be.config.workflowActorType, workflowInstanceID, workflowRevision).
		WithRawDataBytes(requestBytes).
		WithContentType(invokev1.JSONContentType)
.....
}

func NewInvokeMethodRequest(method string) *InvokeMethodRequest {
	return &InvokeMethodRequest{
		r: &internalv1pb.InternalInvokeRequest{
			Ver: DefaultAPIVersion,
			Message: &commonv1pb.InvokeRequest{
				Method: method,
			},
		},
	}
}
```

这样 backend.HistoryEvent 的数据就被以 raw data 的形式在 `commonv1pb.InvokeRequest` 中被传递给 app 了。