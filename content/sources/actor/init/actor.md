---
title: "actor初始化过程"
linkTitle: "actor"
weight: 20
date: 2021-02-24
description: >
  actor 初始化
---

## actor init()

actor 初始化代码在 `pkg/actors/actors.go` 文件中：

```go
func (a *actorsRuntime) Init(ctx context.Context) error {
	if a.closed.Load() {
		return errors.New("actors runtime has already been closed")
	}

	if len(a.actorsConfig.PlacementAddresses) == 0 {
		return errors.New("actors: couldn't connect to placement service: address is empty")
	}

	if len(a.actorsConfig.Config.HostedActorTypes.ListActorTypes()) > 0 {
		if !a.haveCompatibleStorage() {
			return ErrIncompatibleStateStore
		}
	}

	a.actorsReminders.Init(ctx)
	a.timers.Init(ctx)

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

先后执行了 reminder 的初始化，timer 的初始化，然后执行 placement 的初始化。