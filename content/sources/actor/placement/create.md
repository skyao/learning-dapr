---
title: "创建placement"
linkTitle: "创建placement"
weight: 20
date: 2021-02-24
description: >
  创建placement
---

## 创建过程

代码实现在 pkg/actors/placement/placement.go 中：

```go
// NewActorPlacement initializes ActorPlacement for the actor service.
func NewActorPlacement(opts ActorPlacementOpts) internal.PlacementService {
	servers := addDNSResolverPrefix(opts.ServerAddrs)
	return &actorPlacement{
		actorTypes:      opts.ActorTypes,
		appID:           opts.AppID,
		runtimeHostName: opts.RuntimeHostname,
		podName:         opts.PodName,
		serverAddr:      servers,

		client:          newPlacementClient(getGrpcOptsGetter(servers, opts.Security)),
		placementTables: &hashing.ConsistentHashTables{Entries: make(map[string]*hashing.Consistent)},

		unblockSignal:      make(chan struct{}, 1),
		appHealthFn:        opts.AppHealthFn,
		afterTableUpdateFn: opts.AfterTableUpdateFn,
		closeCh:            make(chan struct{}),
		resiliency:         opts.Resiliency,
	}
}
```

actorPlacement 维护 actor 实例的成员资格和一致性哈希表，以便在与 placement service 交互时发现 actor。


