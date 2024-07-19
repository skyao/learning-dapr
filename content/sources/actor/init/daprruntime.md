---
title: "dapr runtime 初始化过程"
linkTitle: "dapr runtime"
weight: 10
date: 2021-02-24
description: >
  dapr actor 初始化是 dapr runtime 初始化的一部分
---

## dapr runtime 初始化

代码在 `dapr/dapr` 仓库的 `pkg/runtime/runtime.go` 中。

dapr runtime 初始化过程中，在第一次 app 变得可用之后，就会继续完成初始化过程：

```go
func (a *DaprRuntime) appHealthChanged(ctx context.Context, status uint8) {

		// First time the app becomes healthy, complete the init process
		if a.appHealthReady != nil {
			if err := a.appHealthReady(ctx); err != nil {
				log.Warnf("Failed to complete app init: %s ", err)
			}
			a.appHealthReady = nil
		}

    ......
```

如果 Actors Enabled，就会执行 initActors() 方法：


```go
// appHealthReadyInit completes the initialization phase and is invoked after the app is healthy
func (a *DaprRuntime) appHealthReadyInit(ctx context.Context) (err error) {
	// Load app configuration (for actors) and init actors
	a.loadAppConfiguration(ctx)

	if a.runtimeConfig.ActorsEnabled() {
		err = a.initActors(ctx)
		if err != nil {
			log.Warn(err)
		} else {
			// Workflow engine depends on actor runtime being initialized
			// This needs to be called before "SetActorsInitDone" on the universal API object to prevent a race condition in workflow methods
			a.initWorkflowEngine(ctx)

			a.daprUniversalAPI.SetActorRuntime(a.actor)
		}
	} else {
		// If actors are not enabled, still invoke SetActorRuntime on the workflow engine with `nil` to unblock startup
		a.workflowEngine.SetActorRuntime(nil)
	}

	// We set actors as initialized whether we have an actors runtime or not
	a.daprUniversalAPI.SetActorsInitDone()

	if cb := a.runtimeConfig.registry.ComponentsCallback(); cb != nil {
		if err = cb(registry.ComponentRegistry{
			DirectMessaging: a.directMessaging,
			CompStore:       a.compStore,
		}); err != nil {
			return fmt.Errorf("failed to register components with callback: %w", err)
		}
	}

	return nil
}
```


```go
func (a *DaprRuntime) initActors(ctx context.Context) error {
	err := actors.ValidateHostEnvironment(a.runtimeConfig.mTLSEnabled, a.runtimeConfig.mode, a.namespace)
	if err != nil {
		return rterrors.NewInit(rterrors.InitFailure, "actors", err)
	}
	a.actorStateStoreLock.Lock()
	defer a.actorStateStoreLock.Unlock()

  // 获取 actor state store 
	actorStateStoreName, ok := a.processor.State().ActorStateStoreName()
	if !ok {
		log.Info("actors: state store is not configured - this is okay for clients but services with hosted actors will fail to initialize!")
	}
	actorConfig := actors.NewConfig(actors.ConfigOpts{
		HostAddress:        a.hostAddress,
		AppID:              a.runtimeConfig.id,
		PlacementAddresses: a.runtimeConfig.placementAddresses,
		Port:               a.runtimeConfig.internalGRPCPort,
		Namespace:          a.namespace,
		AppConfig:          a.appConfig,
		HealthHTTPClient:   a.channels.AppHTTPClient(),
		HealthEndpoint:     a.channels.AppHTTPEndpoint(),
		AppChannelAddress:  a.runtimeConfig.appConnectionConfig.ChannelAddress,
		PodName:            getPodName(),
	})

	act := actors.NewActors(actors.ActorsOpts{
		AppChannel:       a.channels.AppChannel(),
		GRPCConnectionFn: a.grpc.GetGRPCConnection,
		Config:           actorConfig,
		TracingSpec:      a.globalConfig.GetTracingSpec(),
		Resiliency:       a.resiliency,
		StateStoreName:   actorStateStoreName,
		CompStore:        a.compStore,
		// TODO: @joshvanl Remove in Dapr 1.12 when ActorStateTTL is finalized.
		StateTTLEnabled: a.globalConfig.IsFeatureEnabled(config.ActorStateTTL),
		Security:        a.sec,
	})
	err = act.Init(ctx)
	if err == nil {
		a.actor = act
		return nil
	}
	return rterrors.NewInit(rterrors.InitFailure, "actors", err)
}
```

这里能看到有一些参数是必须配置的：

- placementAddresses： 以便和 placement service 交互
- internalGRPCPort：以便 sidecar 可以和其他 sidecar 交互

dapr runtime 创建出 actor 之后，就会调用 actor.init() 方法执行 actor 的初始化。