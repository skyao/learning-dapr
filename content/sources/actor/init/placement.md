---
title: "placement初始化过程"
linkTitle: "placement"
weight: 50
date: 2021-02-24
description: >
  placement 初始化
---

## placement init()

placement 初始化代码在 `pkg/actors/actors.go`:

```go
func (a *actorsRuntime) Init(ctx context.Context) error {
  ......  
  if a.placement == nil {
		a.placement = placement.NewActorPlacement(placement.ActorPlacementOpts{
			ServerAddrs:     a.actorsConfig.Config.PlacementAddresses,
			Security:        a.sec,
			AppID:           a.actorsConfig.Config.AppID,
			RuntimeHostname: a.actorsConfig.GetRuntimeHostname(),
			PodName:         a.actorsConfig.Config.PodName,
			ActorTypes:      a.actorsConfig.Config.HostedActorTypes.ListActorTypes(),
			Resiliency:      a.resiliency,
			AppHealthFn:     a.getAppHealthCheckChan,
			AfterTableUpdateFn: func() {
				a.drainRebalancedActors()
				a.actorsReminders.OnPlacementTablesUpdated(ctx)
			},
		})

		a.placement.SetHaltActorFns(a.haltActor, a.haltAllActors)
		a.placement.SetOnAPILevelUpdate(func(apiLevel uint32) {
			a.apiLevel.Store(apiLevel)
			log.Infof("Actor API level in the cluster has been updated to %d", apiLevel)
		})
	}


	a.wg.Add(1)
	go func() {
		defer a.wg.Done()
		a.placement.Start(ctx)
	}()

	a.wg.Add(1)
	go func() {
		defer a.wg.Done()
		a.deactivationTicker(a.actorsConfig, a.haltActor)
	}()

	log.Infof("Actor runtime started. Actor idle timeout: %v. Actor scan interval: %v",
		a.actorsConfig.Config.ActorIdleTimeout, a.actorsConfig.Config.ActorDeactivationScanInterval)

	return nil
  }
```

其中：

- RuntimeHostname 是类似 "192.168.0.101:32003" 这样的值，host ip + dapr internal port。

注意 AfterTableUpdateFn 的设置：

```go
			AfterTableUpdateFn: func() {
				a.drainRebalancedActors()
				a.actorsReminders.OnPlacementTablesUpdated(ctx)
			},
```

还有 SetOnAPILevelUpdate 的设置：

```go
		a.placement.SetOnAPILevelUpdate(func(apiLevel uint32) {
			a.apiLevel.Store(apiLevel)
			log.Infof("Actor API level in the cluster has been updated to %d", apiLevel)
		})
```

然后异步继续执行 `a.placement.Start(ctx)`。
