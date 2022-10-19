---
title: "mdns命名解析"
linkTitle: "mdns"
weight: 100
date: 2021-03-17
description: >
  mdns命名解析实现
---

### 基本输入输出

跳过细节和错误处理，尤其是去除所有同步保护代码（很复杂），只简单看输入和输出：

```go
// ResolveID 通过 mDNS 将名称解析为地址。
func (m *Resolver) ResolveID(req nameresolution.ResolveRequest) (string, error) {
	m.browseOne(ctx, req.ID, published)

	select {
	case addr := <-sub.AddrChan:
		return addr, nil
	case err := <-sub.ErrChan:
		return "", err
	case <-time.After(subscriberTimeout):
		return "", fmt.Errorf("timeout waiting for address for app id %s", req.ID)
	}
}

func (m *Resolver) browseOne(ctx context.Context, appID string, published chan struct{}) {
  err := m.browse(browseCtx, appID, onFirst)
}
```

注意：只用到了 req.ID, 全程没有使用 req.Namespace，也就是 MDNS 根本不支持 Namespace.

### mdns解析方式

mdns 的核心实现在 browseOne() 方法中：

```go
func (m *Resolver) browseOne(ctx context.Context, appID string, published chan struct{}) {
  // 启动一个 goroutine 异步执行
	go func() {
		var addr string

		browseCtx, cancel := context.WithCancel(ctx)
		defer cancel()

    // 准备回调函数，收到第一个地址之后就取消 browse，所以这个函数名为 browseOne
		onFirst := func(ip string) {
			addr = ip
			cancel() // cancel to stop browsing.
		}

		m.logger.Debugf("Browsing for first mDNS address for app id %s", appID)

    // 执行 browse
		err := m.browse(browseCtx, appID, onFirst)
		// 忽略错误处理
    ......
    
		m.pubAddrToSubs(appID, addr)

		published <- struct{}{} // signal that all subscribers have been notified.
	}()
}
```

继续看 browse 的实现：

```go
// browse 将对所提供的 App ID 进行无阻塞的 mdns 网络浏览
func (m *Resolver) browse(ctx context.Context, appID string, onEach func(ip string)) error {
  ......
}
```

首先通过 zeroconf.NewResolver 构建一个 Resolver：

```go
  import "github.com/grandcat/zeroconf"

	resolver, err := zeroconf.NewResolver(nil)
  if err != nil {
		return fmt.Errorf("failed to initialize resolver: %w", err)
	}
  ......
```

zeroconf  是一个纯Golang库，采用多播 DNS-SD 来浏览和解析网络中的服务，并在本地网络中注册自己的服务。

执行mdns解析的代码是 resolver.Browse() 方法，解析的结果会异步发送到 entries 这个 channel 中：

```go
	entries := make(chan *zeroconf.ServiceEntry)	
  if err = resolver.Browse(ctx, appID, "local.", entries); err != nil {
		return fmt.Errorf("failed to browse: %w", err)
	}
```

每个从 mDNS browse 返回的 service entry 会这样处理：

```go
	// handle each service entry returned from the mDNS browse.
	go func(results <-chan *zeroconf.ServiceEntry) {
		for {
			select {
			case entry := <-results:
				if entry == nil {
					break
				}
        // 调用 handleEntry 方法来处理每个返回的 service entry
				handleEntry(entry)
			case <-ctx.Done():
        // 如果所有 service entry 都处理完成了，或者是出错（取消或者超时）
        // 此时需要推出 browse，但在退出之前需要检查一下是否有已经收到但还没有处理的结果
				for len(results) > 0 {
					handleEntry(<-results)
				}

				if errors.Is(ctx.Err(), context.Canceled) {
					m.logger.Debugf("mDNS browse for app id %s canceled.", appID)
				} else if errors.Is(ctx.Err(), context.DeadlineExceeded) {
					m.logger.Debugf("mDNS browse for app id %s timed out.", appID)
				}

				return // stop listening for results.
			}
		}
	}(entries)
```

handleEntry() 方法的实现：

```go
	handleEntry := func(entry *zeroconf.ServiceEntry) {
		for _, text := range entry.Text {
      // 检查appID看是否是自己要查找的app
			if text != appID {
				m.logger.Debugf("mDNS response doesn't match app id %s, skipping.", appID)
				break
			}

			m.logger.Debugf("mDNS response for app id %s received.", appID)

      // 检查是否有 IPv4 或者 ipv6 地址
			hasIPv4Address := len(entry.AddrIPv4) > 0
			hasIPv6Address := len(entry.AddrIPv6) > 0

			if !hasIPv4Address && !hasIPv6Address {
				m.logger.Debugf("mDNS response for app id %s doesn't contain any IPv4 or IPv6 addresses, skipping.", appID)
				break
			}

			var addr string
			port := entry.Port
      // 目前只支持取第一个地址
			// TODO: we currently only use the first IPv4 and IPv6 address.
			// We should understand the cases in which additional addresses
			// are returned and whether we need to support them.
      // 加入到缓存中，缓存后面细看
			if hasIPv4Address {
				addr = fmt.Sprintf("%s:%d", entry.AddrIPv4[0].String(), port)
				m.addAppAddressIPv4(appID, addr)
			}
			if hasIPv6Address {
				addr = fmt.Sprintf("%s:%d", entry.AddrIPv6[0].String(), port)
				m.addAppAddressIPv6(appID, addr)
			}

      // 开始回调，就是前面说的拿到第一个地址就取消 browse
			if onEach != nil {
				onEach(addr) // invoke callback.
			}
		}
	}
```

至此就完成了 mdns 的解析，从 ID 到 address。

### 缓存设计

mdns 是非常慢的，为了性能就需要缓存解析后的地址，前面的代码在解析完成之后会保存这些地址：

```go
// addAppAddressIPv4 adds an IPv4 address to the
// cache for the provided app id.
func (m *Resolver) addAppAddressIPv4(appID string, addr string) {
	m.ipv4Mu.Lock()
	defer m.ipv4Mu.Unlock()

	m.logger.Debugf("Adding IPv4 address %s for app id %s cache entry.", addr, appID)
	if _, ok := m.appAddressesIPv4[appID]; !ok {
		var addrList addressList
		m.appAddressesIPv4[appID] = &addrList
	}
	m.appAddressesIPv4[appID].add(addr)
}
```

在解析之前，在 ResolveID() 方法中会线尝试检查缓存中是否有数据，如果有就直接使用：

```go
func (m *Resolver) ResolveID(req nameresolution.ResolveRequest) (string, error) {
	// check for cached IPv4 addresses for this app id first.
	if addr := m.nextIPv4Address(req.ID); addr != nil {
		return *addr, nil
	}

	// check for cached IPv6 addresses for this app id second.
	if addr := m.nextIPv6Address(req.ID); addr != nil {
		return *addr, nil
	}
  ......
}
```

从缓存中获取appID对应的地址：

```go
// nextIPv4Address returns the next IPv4 address for
// the provided app id from the cache.
func (m *Resolver) nextIPv4Address(appID string) *string {
	m.ipv4Mu.RLock()
	defer m.ipv4Mu.RUnlock()
	addrList, exists := m.appAddressesIPv4[appID]
	if exists {
		addr := addrList.next()
		if addr != nil {
			m.logger.Debugf("found mDNS IPv4 address in cache: %s", *addr)

			return addr
		}
	}

	return nil
}
```

addrList.next() 比较有意思，这里不是要获取地址列表，而是取单个地址。也就是说，当有多个地址时，这里 addrList.next() 实际上实现了负载均衡 ^0^ 

### 负载均衡

addressList 结构体的组成：

```go
// addressList represents a set of addresses along with
// data used to control and access said addresses.
type addressList struct {
	addresses []address
	counter   int
	mu        sync.RWMutex
}
```

除了地址数组之外，还有一个 counter ，以及并发保护的读写锁。

```go
// max integer value supported on this architecture.
const maxInt = int(^uint(0) >> 1)

// next 从列表中获取下一个地址，考虑到当前的循环实现。除了尽力而为的线性迭代，对选择没有任何保证。
func (a *addressList) next() *string {
  // 获取读锁
	a.mu.RLock()
	defer a.mu.RUnlock()

	if len(a.addresses) == 0 {
		return nil
	}
  // 如果 counter 达到 maxInt，就从头再来
	if a.counter == maxInt {
		a.counter = 0
	}
  // 用地址数量 对 counter 求余，去余数所对应的地址，然后counter递增
  // 相当于一个最简单常见的 轮询 算法
	index := a.counter % len(a.addresses)
	addr := a.addresses[index]
	a.counter++

	return &addr.ip
}
```

### 并发保护

为了避免多个请求同时去解析同一个 ID，因此设计了并发保护机制，对于单个ID，只容许一个请求执行解析，其他请求会等待这个解析的结果：

```go

// ResolveID resolves name to address via mDNS.
func (m *Resolver) ResolveID(req nameresolution.ResolveRequest) (string, error) {

	sub := NewSubscriber()

	// add the sub to the pool of subs for this app id.
	m.subMu.Lock()
	appIDSubs, exists := m.subs[req.ID]
	if !exists {
		// WARN: must set appIDSubs variable for use below.
		appIDSubs = NewSubscriberPool(sub)
		m.subs[req.ID] = appIDSubs
	} else {
		appIDSubs.Add(sub)
	}
	m.subMu.Unlock()

	// only one subscriber per pool will perform the first browse for the
	// requested app id. The rest will subscribe for an address or error.
	var once *sync.Once
	var published chan struct{}
	ctx, cancel := context.WithTimeout(context.Background(), browseOneTimeout)
	defer cancel()
	appIDSubs.Once.Do(func() {
		published = make(chan struct{})
		m.browseOne(ctx, req.ID, published)

		// once will only be set for the first browser.
		once = new(sync.Once)
	})
	......
}
```

### 总结

mdns name resolver 返回的是一个简单的 ip 地址+端口（v4或者v6），形如 "192.168.0.100:8000"。
