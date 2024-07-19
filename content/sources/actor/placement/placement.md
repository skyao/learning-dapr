---
title: "placement定义"
linkTitle: "placement定义"
weight: 10
date: 2021-02-24
description: >
  Dapr actor构建块 placement lookup 源码分析
---


## placement 接口定义

PlacementService 的接口定义在文件 `pkg/actors/internal/placement_service.go` 中：

```go
type PlacementService interface {
	io.Closer

	Start(context.Context) error
	WaitUntilReady(ctx context.Context) error
	LookupActor(ctx context.Context, req LookupActorRequest) (LookupActorResponse, error)
	AddHostedActorType(actorType string, idleTimeout time.Duration) error
	ReportActorDeactivation(ctx context.Context, actorType, actorID string) error

	SetHaltActorFns(haltFn HaltActorFn, haltAllFn HaltAllActorsFn)
	SetOnAPILevelUpdate(fn func(apiLevel uint32))
	SetOnTableUpdateFn(fn func())

	// PlacementHealthy returns true if the placement service is healthy.
	PlacementHealthy() bool
	// StatusMessage returns a custom status message.
	StatusMessage() string
}
```

## placement 结构体定义

代码实现在 pkg/actors/placement/placement.go 中：

```go
// actorPlacement maintains membership of actor instances and consistent hash
// tables to discover the actor while interacting with Placement service.
type actorPlacement struct {
	actorTypes []string
	appID      string
	// runtimeHostname is the address and port of the runtime
	runtimeHostName string
	// name of the pod hosting the actor
	podName string

	// client is the placement client.
	client *placementClient

	// serverAddr is the list of placement addresses.
	serverAddr []string
	// serverIndex is the current index of placement servers in serverAddr.
	serverIndex atomic.Int32

	// placementTables is the consistent hashing table map to
	// look up Dapr runtime host address to locate actor.
	placementTables *hashing.ConsistentHashTables
	// placementTableLock is the lock for placementTables.
	placementTableLock sync.RWMutex
	// hasPlacementTablesCh is closed when the placement tables have been received.
	hasPlacementTablesCh chan struct{}

	// apiLevel is the current API level of the cluster
	apiLevel uint32
	// onAPILevelUpdate is invoked when the API level is updated
	onAPILevelUpdate func(apiLevel uint32)

	// unblockSignal is the channel to unblock table locking.
	unblockSignal chan struct{}
	// operationUpdateLock is the lock for three stage commit.
	operationUpdateLock sync.Mutex

	// appHealthFn returns the appHealthCh
	appHealthFn func(ctx context.Context) <-chan bool
	// appHealthy contains the result of the app health checks.
	appHealthy atomic.Bool
	// afterTableUpdateFn is function for post processing done after table updates,
	// such as draining actors and resetting reminders.
	afterTableUpdateFn func()

	// callback invoked to halt all active actors
	haltAllActorsFn internal.HaltAllActorsFn

	// shutdown is the flag when runtime is being shutdown.
	shutdown atomic.Bool
	// shutdownConnLoop is the wait group to wait until all connection loop are done
	shutdownConnLoop sync.WaitGroup
	// closeCh is the channel to close the placement service.
	closeCh chan struct{}

	resiliency resiliency.Provider
}
```

actorPlacement 维护 actor 实例的成员资格和一致性哈希表，以便在与 placement service 交互时发现 actor。



### AddHostedActorType()

通过将 actor 类型添加到已知 actor 类型列表中来注册 actor类型（如果尚未注册），当下一次心跳启动时，place table 将得到更新:

```go

type actorPlacement struct {
	actorTypes []string
	......
}

func (p *actorPlacement) AddHostedActorType(actorType string, idleTimeout time.Duration) error {
	for _, t := range p.actorTypes {
		if t == actorType {
			return fmt.Errorf("actor type %s already registered", actorType)
		}
	}

	p.actorTypes = append(p.actorTypes, actorType)
	return nil
}
```