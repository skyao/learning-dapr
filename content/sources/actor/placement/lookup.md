---
title: "lookup"
linkTitle: "lookup"
weight: 40
date: 2021-02-24
description: >
  Dapr actor构建块 placement lookup 源码分析
---

## placement lookup

代码实现在 `pkg/actors/placement/placement.go` 中：

```go
// LookupActor 使用一致性的散列表解析 actor 服务实例地址。
func (p *actorPlacement) LookupActor(ctx context.Context, req internal.LookupActorRequest) (internal.LookupActorResponse, error) {
	// 在此重试，以允许 placement table 传播/重新平衡。
	policyDef := p.resiliency.BuiltInPolicy(resiliency.BuiltInActorNotFoundRetries)
	policyRunner := resiliency.NewRunner[internal.LookupActorResponse](ctx, policyDef)
	return policyRunner(func(ctx context.Context) (res internal.LookupActorResponse, rErr error) {
		// 查找 actor
		rAddr, rAppID, rErr := p.doLookupActor(ctx, req.ActorType, req.ActorID)
		if rErr != nil {
			return res, fmt.Errorf("error finding address for actor %s/%s: %w", req.ActorType, req.ActorID, rErr)
		} else if rAddr == "" {
			return res, fmt.Errorf("did not find address for actor %s/%s", req.ActorType, req.ActorID)
		}
		res.Address = rAddr
		res.AppID = rAppID
		return res, nil
	})
}
```

关键的 doLookupActor() 方法的实现:

```go
func (p *actorPlacement) doLookupActor(ctx context.Context, actorType, actorID string) (string, string, error) {
	// 对于 workflow：
	// actorType = "dapr.internal.default.WorkflowConsoleApp.workflow"
	// actorID = “"460a3798-be81-477e-add4-ee5a2fd23ba7"” 这是 workflow 的 instance id
  // 加读锁
	p.placementTableLock.RLock()
	defer p.placementTableLock.RUnlock()

	if p.placementTables == nil {
		return "", "", errors.New("placement tables are not set")
	}

  // 先根据 actorType 找到符合要求的 Entries
	// 上一节 placement start 时看到 p.placementTables.Entries 的 key 是 actor type
	// 这里根据 actor type 找到保存的 entry 信息
	// 由于 workflow 的 actor type 是固定的 "dapr.internal.default.WorkflowConsoleApp.workflow"
	// 所以这里的 t 总是返回 workfflow 相关的信息
	t := p.placementTables.Entries[actorType]
	if t == nil {
		// 如果没有找到，返回空
		return "", "", nil
	}

	// 找到了entry，t 的类型是 *Consistent
	host, err := t.GetHost(actorID)
	if err != nil || host == nil {
		return "", "", nil //nolint:nilerr
	}
	return host.Name, host.AppID, nil
}
```

p.placementTables 的结构体定义如下：

```go
type ConsistentHashTables struct {
	Version string
	Entries map[string]*Consistent
}
```

所以 LookupActor() 可以简化为两个步骤：

1. 根据key找到consistent：逻辑非常简单，每个 actor 都有自己的唯一的 actor type，用 map 保存起来，key 就是 actor type。
2. 根据 actorID 在 consistent 中找到 host 等实例信息：这就要看 consistent 的实现细节了。

## consistent_hash 的实现

### Consistent 的结构体

Consistent 的结构体定义如下：

```go
// Consistent 表示一种用于一致性hash的数据结构。
type Consistent struct {
	hosts             map[uint64]string
	sortedSet         []uint64
	loadMap           map[string]*Host
	totalLoad         int64
	replicationFactor int

	sync.RWMutex
}
```

### Consistent lookup

`host, err := t.GetHost(actorID)` 代码对应的 GetHost() 方法：

```go
func (c *Consistent) GetHost(key string) (*Host, error) {
	// 这个 key 是 actorID
	h, err := c.Get(key)
	if err != nil {
		return nil, err
	}

	return c.loadMap[h], nil
}
```



```go
// Get 返回拥有 `key` 的 host。
//
// As described in https://en.wikipedia.org/wiki/Consistent_hashing
//
// It returns ErrNoHosts if the ring has no hosts in it.
func (c *Consistent) Get(key string) (string, error) {
	// 加读锁
	c.RLock()
	defer c.RUnlock()

  // 如果 hosts 这个 map[uint64]string 中没有数据，则只能返回找不到
	if len(c.hosts) == 0 {
		return "", ErrNoHosts
	}

  // 对 key 进行 hash
	h := c.hash(key)
	// 然后进行查找
	idx := c.search(h)
	// 根据 idx 在 sortedSet 中去下标，再返回 hosts 的数据
	return c.hosts[c.sortedSet[idx]], nil
}
```

search() 方法的实现：

```go
func (c *Consistent) search(key uint64) int {
	idx := sort.Search(len(c.sortedSet), func(i int) bool {
		return c.sortedSet[i] >= key
	})

	if idx >= len(c.sortedSet) {
		idx = 0
	}
	return idx
}
```