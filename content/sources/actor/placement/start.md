---
title: "start placement"
linkTitle: "start placement"
weight: 30
date: 2021-02-24
description: >
  start placement
---

## start 实现

启动连接安置服务，注册成为会员，并发送心跳信息定期报告当前成员状态。

### 重置状态

有几个状态是 atomic 类型，先重置：

```go
type actorPlacement struct {
	serverIndex atomic.Int32
	appHealthy atomic.Bool
  shutdown atomic.Bool
  ......
}

func (p *actorPlacement) Start(ctx context.Context) error {
  // 设置 serverIndex = 0
	p.serverIndex.Store(0)
  // 设置 shutdown 为 false ： 刚启动，当然应该为false；停止后再启动，重置为 false
	p.shutdown.Store(false)
  // 设置 appHealthy 为 false 
	p.appHealthy.Store(true)
	p.resetPlacementTables()
  ......
```

resetPlacementTables() 重置 placement table：



```go
type actorPlacement struct {
  // placementTables 是一致性哈希表映射，用于 查找 Dapr runtime host 地址以定位 actor。
	placementTables *hashing.ConsistentHashTables
	// placementTableLock 是 placementTables 的锁。这是一个读写锁
	placementTableLock sync.RWMutex
  // 当收到 placement tables 时，hasPlacementTablesCh 关闭。
	hasPlacementTablesCh chan struct{}
  ......
}

// ConsistentHashTables 是一个表，保存着具有给定版本的一致性哈希值的映射。
type ConsistentHashTables struct {
	Version string
	Entries map[string]*Consistent
}

func (p *actorPlacement) resetPlacementTables() {
  // 如果 hasPlacementTablesCh 不为空，则关闭这个 channel
	if p.hasPlacementTablesCh != nil {
		close(p.hasPlacementTablesCh)
	}

  // 重新创建一个新的 channel
	p.hasPlacementTablesCh = make(chan struct{})
  // 清空 p.placementTables 
	maps.Clear(p.placementTables.Entries)
  // 重置 placementTables.Version 为空
	p.placementTables.Version = ""
}
```

### 建立流连接

下一步，开始建立流连接：

```go
	if !p.establishStreamConn(ctx) {
		return nil
	}
```

establishStreamConn() 的内容很长

```go
// Interval to wait for app health's readiness
const placementReadinessWaitInterval = 500 * time.Millisecond

func (p *actorPlacement) establishStreamConn(ctx context.Context) (established bool) {
  // 发生错误时重新连接的回退
	bo := backoff.NewExponentialBackOff()
	bo.InitialInterval = placementReconnectMinInterval
	bo.MaxInterval = placementReconnectMaxInterval
	bo.MaxElapsedTime = 0 // Retry forever

	logFailureShown := false

  // 循环直到收到关闭的要求（p.shutdown 被设置为true）
	for !p.shutdown.Load() {
    // 如果上下文被取消，则不再重试连接
		if ctx.Err() != nil {
			return false
		}

    // 在应用程序恢复健康之前，停止重新连接 placement。
		if !p.appHealthy.Load() {
      // 我们在这里不使用指数后退，因为我们还没有开始建立连接
      // sleep 500 毫秒之后重新尝试
			time.Sleep(placementReadinessWaitInterval)
			continue
		}

    // 睡眠后再次检查上下文的有效性
    // 防备 sleep 之后 context 被设置为取消
		if ctx.Err() != nil {
			return false
		}

    // 获取当前 serverIndex 所指向的 serverAddr
    // 前面 serverIndex 被重置为 0 了
		serverAddr := p.serverAddr[p.serverIndex.Load()]

		if !logFailureShown {
			log.Debug("try to connect to placement service: " + serverAddr)
		}

    // 连接到 placement 服务器端
		err := p.client.connectToServer(ctx, serverAddr)
		if err == errEstablishingTLSConn {
      // 如果是 errEstablishingTLSConn 失败则返回
			return false
		}

		if err != nil {
      // 如果是其他错误
			if !logFailureShown {
				log.Debugf("Error connecting to placement service (will retry to connect in background): %v", err)
				// Don't show the debug log more than once per each reconnection attempt
				logFailureShown = true
			}

      // 尝试不同的 placemnent 服务实例
      // serverIndex+1，取模的作用是达到最后一个时自动设置为0
			p.serverIndex.Store((p.serverIndex.Load() + 1) % int32(len(p.serverAddr)))

      // 停止所有运行中的 actor，然后重置 placement tables
			if p.haltAllActorsFn != nil {
				p.haltAllActorsFn()
			}
			p.resetPlacementTables()

      // 采用指数回退的睡眠模式
			select {
			case <-time.After(bo.NextBackOff()):
			case <-ctx.Done():
				return false
			}

      // 失败则等待一段时间之后继续
			continue
		}

    // 如果能连接成功
		log.Debug("Established connection to placement service at " + p.client.clientConn.Target())
		return true
	}

	return false
}
```

### shutdownConnLoop for close

等待 ctx 或者 closeCh 要求关闭

```go
type actorPlacement struct {
  // shutdownConnLoop 是 wait group，用于等待所有连接循环结束
	shutdownConnLoop sync.WaitGroup
  ......
}

	ctx, cancel := context.WithCancel(ctx)
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()

		select {
		case <-ctx.Done():
		case <-p.closeCh:
		}
		cancel()
	}()
```

### shutdownConnLoop for appHealth

```go
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()
		ch := p.appHealthFn(ctx)
		if ch == nil {
			return
		}

		for healthy := range ch {
			p.appHealthy.Store(healthy)
		}
	}()
```

### shutdownConnLoop for placement disconnect

```go
  // 建立连接循环，每当断开连接时，它就会开始运行，试图连接到新的服务器。
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()
		for !p.shutdown.Load() {
			// wait until disconnection occurs or shutdown is triggered
			p.client.waitUntil(func(streamConnAlive bool) bool {
				return !streamConnAlive || p.shutdown.Load()
			})

			if p.shutdown.Load() {
				break
			}
			p.establishStreamConn(ctx)
		}
	}()
```

### shutdownConnLoop for placement table update

```go
  // 建立接收通道，检索 placement table 更新
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()
		for !p.shutdown.Load() {
			// Wait until stream is connected or shutdown is triggered.
			p.client.waitUntil(func(streamAlive bool) bool {
				return streamAlive || p.shutdown.Load()
			})

      // 调用 placementClient client 的 recv()
			resp, err := p.client.recv()
			if p.shutdown.Load() {
				break
			}

			// TODO: we may need to handle specific errors later.
			if err != nil {
				p.client.disconnectFn(func() {
					p.onPlacementError(err) // depending on the returned error a new server could be used
				})
			} else {
        // 终于可以更新 placement table 了
				p.onPlacementOrder(resp)
			}
		}
	}()
```

### 更新 placement 的实现

```go
type PlacementOrder struct {
	Tables    *PlacementTables `protobuf:"bytes,1,opt,name=tables,proto3" json:"tables,omitempty"`
	Operation string           `protobuf:"bytes,2,opt,name=operation,proto3" json:"operation,omitempty"`
}

type actorPlacement struct {
  // operationUpdateLock 是三阶段提交的锁。
	operationUpdateLock sync.Mutex
  ......
}

func (p *actorPlacement) onPlacementOrder(in *v1pb.PlacementOrder) {
	log.Debugf("Placement order received: %s", in.GetOperation())
	diag.DefaultMonitoring.ActorPlacementTableOperationReceived(in.GetOperation())

  // 当更新的表到达时，锁定所有调用
	p.operationUpdateLock.Lock()
	defer p.operationUpdateLock.Unlock()

	switch in.GetOperation() {
	case lockOperation:
		p.blockPlacements()

		go func() {
			// TODO: Use lock-free table update.
			// current implementation is distributed two-phase locking algorithm.
			// If placement experiences intermittently outage during updateplacement,
			// user application will face 5 second blocking even if it can avoid deadlock.
			// It can impact the entire system.
			time.Sleep(time.Second * 5)
			p.unblockPlacements()
		}()

	case unlockOperation:
		p.unblockPlacements()

	case updateOperation:
		p.updatePlacements(in.GetTables())
	}
}
```

p.updatePlacements() 的实现：

```go

type PlacementTable struct {
  ......
	Hosts     map[uint64]string 
	SortedSet []uint64          
	LoadMap   map[string]*Host  
	TotalLoad int64             
}

func (p *actorPlacement) updatePlacements(in *v1pb.PlacementTables) {
	updated := false
	var updatedAPILevel *uint32
	func() {
    // 先加锁
		p.placementTableLock.Lock()
		defer p.placementTableLock.Unlock()

    // 如果已经是这个版本了，就不用更次更新了
		if in.GetVersion() == p.placementTables.Version {
			return
		}

    // 如果是输入的 api level 和当前的 api level 不一致，更新 api level
		if in.GetApiLevel() != p.apiLevel {
			p.apiLevel = in.GetApiLevel()
			updatedAPILevel = ptr.Of(in.GetApiLevel())
		}

    // 开始更新前的清理： 清理 placementTables Entries 
		maps.Clear(p.placementTables.Entries)
    // 获取并设置 version： 是不是应该最后再设置？
    // 万一下面的更新次操作出问题，version 没有更改的话还有机会下一次继续更新
		p.placementTables.Version = in.GetVersion()
    // 游历输入的每个 entry，GetEntries 的类型是 map[string]*PlacementTable 
		for k, v := range in.GetEntries() {
      // 这里 k 是 string，v 是 *PlacementTable 
      // 准备把输入的 v 的 loadmap 转为 
      // 输入的 loadmap 类型是 LoadMap   map[string]*Host
      // 本地 loadMap 的类型是 map[string]*hashing.Host， value 的类型不一样
			loadMap := make(map[string]*hashing.Host, len(v.GetLoadMap()))
			for lk, lv := range v.GetLoadMap() {
        // lk 是 key，string 类型，继续作为本地 loadMap 的 key
        // lv 是 value，*Host 类型，要转为本地的 Host
				loadMap[lk] = hashing.NewHost(lv.GetName(), lv.GetId(), lv.GetLoad(), lv.GetPort())
			}
      // loadMap 转换完成，更新 placementTables
			p.placementTables.Entries[k] = hashing.NewFromExisting(v.GetHosts(), v.GetSortedSet(), loadMap)
		}

    // 标记为已经更新
		updated = true
		if p.hasPlacementTablesCh != nil {
			close(p.hasPlacementTablesCh)
			p.hasPlacementTablesCh = nil
		}
	}()

	if updatedAPILevel != nil && p.onAPILevelUpdate != nil {
		p.onAPILevelUpdate(*updatedAPILevel)
	}

	if updated {
		// May call LookupActor inside, so should not do this with placementTableLock locked.
		if p.afterTableUpdateFn != nil {
			p.afterTableUpdateFn()
		}
		log.Infof("Placement tables updated, version: %s", in.GetVersion())
	}
}
```

仔细对照看 host 转换的逻辑：

```go
// 输入的 Host 类型，protobuf
type Host struct {
  ......
	Name     string   
	Port     int64    
	Load     int64    
	Entities []string 
	Id       string   
	Pod      string   
	// Version of the Actor APIs supported by the Dapr runtime
	ApiLevel uint32 
}

loadMap[lk] = hashing.NewHost(lv.GetName(), lv.GetId(), lv.GetLoad(), lv.GetPort())

// Host 表示有状态实体的主机，具有给定的名称、ID、端口和负载。
// 类型基本匹配
type Host struct {
	Name  string
	Port  int64
	Load  int64
	AppID string
}
```

NewFromExisting() 方法创建 Consistent

```go
type PlacementTable struct {
  ......
	Hosts     map[uint64]string 
	SortedSet []uint64          
	LoadMap   map[string]*Host  
	TotalLoad int64             
}

p.placementTables.Entries[k] = hashing.NewFromExisting(v.GetHosts(), v.GetSortedSet(), loadMap)

// NewFromExisting creates a new consistent hash from existing values.
func NewFromExisting(hosts map[uint64]string, sortedSet []uint64, loadMap map[string]*Host) *Consistent {
	return &Consistent{
		hosts:     hosts,
		sortedSet: sortedSet,
		loadMap:   loadMap,
	}
}
```

hosts 和 sortedSet 不需要转换直接保存，loadmap 按照上面的逻辑转了一下类型。


```go
	// Send the current host status to placement to register the member and
	// maintain the status of member by placement.
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()
		for !p.shutdown.Load() {
			// Wait until stream is connected or shutdown is triggered.
			p.client.waitUntil(func(streamAlive bool) bool {
				return streamAlive || p.shutdown.Load()
			})

			if p.shutdown.Load() {
				break
			}

			// appHealthy is the health status of actor service application. This allows placement to update
			// the member list and hashing table quickly.
			if !p.appHealthy.Load() {
				// app is unresponsive, close the stream and disconnect from the placement service.
				// Then Placement will remove this host from the member list.
				log.Debug("Disconnecting from placement service by the unhealthy app")

				p.client.disconnect()
				p.placementTableLock.Lock()
				p.resetPlacementTables()
				p.placementTableLock.Unlock()
				if p.haltAllActorsFn != nil {
					haltErr := p.haltAllActorsFn()
					if haltErr != nil {
						log.Errorf("Failed to deactivate all actors: %v", haltErr)
					}
				}
				continue
			}

			host := v1pb.Host{
				Name:     p.runtimeHostName,
				Entities: p.actorTypes,
				Id:       p.appID,
				Load:     1, // Not used yet
				Pod:      p.podName,
				// Port is redundant because Name should include port number
				// Port: 0,
				ApiLevel: internal.ActorAPILevel,
			}

			err := p.client.send(&host)
			if err != nil {
				diag.DefaultMonitoring.ActorStatusReportFailed("send", "status")
				log.Errorf("Failed to report status to placement service : %v", err)
			}

			// No delay if stream connection is not alive.
			if p.client.isConnected() {
				diag.DefaultMonitoring.ActorStatusReported("send")
				time.Sleep(statusReportHeartbeatInterval)
			}
		}
	}()

	return nil
}
```

### 上报当前 host 的状态给 placement

```go
  // 将当前主机状态发送给安置，以注册成员，并通过 placement 保持成员状态。
	p.shutdownConnLoop.Add(1)
	go func() {
		defer p.shutdownConnLoop.Done()
		for !p.shutdown.Load() {
			// Wait until stream is connected or shutdown is triggered.
			p.client.waitUntil(func(streamAlive bool) bool {
				return streamAlive || p.shutdown.Load()
			})

			if p.shutdown.Load() {
				break
			}

			// appHealthy is the health status of actor service application. This allows placement to update
			// the member list and hashing table quickly.
			if !p.appHealthy.Load() {
				// app is unresponsive, close the stream and disconnect from the placement service.
				// Then Placement will remove this host from the member list.
				log.Debug("Disconnecting from placement service by the unhealthy app")

				p.client.disconnect()
				p.placementTableLock.Lock()
				p.resetPlacementTables()
				p.placementTableLock.Unlock()
				if p.haltAllActorsFn != nil {
					haltErr := p.haltAllActorsFn()
					if haltErr != nil {
						log.Errorf("Failed to deactivate all actors: %v", haltErr)
					}
				}
				continue
			}

      // 组装要上报的 host 信息
			host := v1pb.Host{
        // name 是 runtime 所在的 hostname （加端口）
				Name:     p.runtimeHostName,
        // p.actorTypes 的类型是 []string， host 的 entity 也是 actor type
				// 在连上 app 之前，这里是空的
				// 在连上 app 之后，如果 app 开启了 workflow，这里就会有两个 actor type
				// 1. "dapr.internal.default.WorkflowConsoleApp.activity"
				// 2. "dapr.internal.default.WorkflowConsoleApp.workflow"
				Entities: p.actorTypes,
        // host 的 id 指的是 appID
				Id:       p.appID,
				Load:     1, // Not used yet
				Pod:      p.podName,
				// Port is redundant because Name should include port number
				// Port: 0,
				ApiLevel: internal.ActorAPILevel,
			}

      // 上报
			err := p.client.send(&host)
			if err != nil {
				diag.DefaultMonitoring.ActorStatusReportFailed("send", "status")
				log.Errorf("Failed to report status to placement service : %v", err)
			}

			// No delay if stream connection is not alive.
			if p.client.isConnected() {
				diag.DefaultMonitoring.ActorStatusReported("send")
				time.Sleep(statusReportHeartbeatInterval)
			}
		}
	}()
```