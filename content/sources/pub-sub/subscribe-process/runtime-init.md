---
type: docs
title: "订阅相关的Runtime初始化"
linkTitle: "Runtime初始化"
weight: 20
date: 2022-04-21
description: >
  Dapr Runtime中和订阅相关的初始化流程
---

在 dapr runtime 启动进行初始化时，需要

- 访问应用以获取应用的订阅信息：比如应用订阅了哪些topic
- 根据配置文件启动 subscribe component 以便连接到外部 message broker 进行订阅
- 将订阅更新的 event 转发给应用

## Dapr runtime初始化component列表

dapr runtime 初始化时会创建和 app 的连接，称为 app channel，然后开始发布订阅的初始化：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
	......
    // 有一个单独的 go routine 负责处理 component 的初始化
    go a.processComponents()
    err = a.loadComponents(opts)
    
	// 等待应用ready： 前提是设置了 app port
	a.blockUntilAppIsReady()

	// 创建 app channel
	err = a.createAppChannel()
    // app channel 支持 http 和 grpc
	a.daprHTTPAPI.SetAppChannel(a.appChannel)
	grpcAPI.SetAppChannel(a.appChannel)
    ......
    
    // 开始发布订阅的初始化
    a.startSubscribing()
}
```

这里有一段复杂的并行初始化components并处理相互依赖的逻辑，忽略这些细节，只看执行 component 初始化的代码：

```go
func (a *DaprRuntime) doProcessOneComponent(category ComponentCategory, comp components_v1alpha1.Component) error {
	switch category {
	case pubsubComponent:
		return a.initPubSub(comp)
	......
	}
	return nil
}

func (a *DaprRuntime) initPubSub(c components_v1alpha1.Component) error {
	pubSub, err := a.pubSubRegistry.Create(c.Spec.Type, c.Spec.Version)

    // 初始化 pubSub component
	err = pubSub.Init(pubsub.Metadata{
		Properties: properties,
	})

	pubsubName := c.ObjectMeta.Name
	a.pubSubs[pubsubName] = pubSub
	return nil
}
```

这个执行完成之后，a.pubSubs 中便保存有当前配置并初始化好的 pubsub 组件列表。

## pubsub组件启动

订阅的初始化在 dapr runtime 启动过程的最后阶段

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    ......
    // 开始发布订阅的初始化
    a.startSubscribing()
}
```

startSubscribing() 方法逐个处理 pubSub 组件:

```go
func (a *DaprRuntime) startSubscribing() {
	for name, pubsub := range a.pubSubs {
		if err := a.beginPubSub(name, pubsub); err != nil {
			log.Errorf("error occurred while beginning pubsub %s: %s", name, err)
		}
	}
}
```

beginPubSub 方法做了两个事情： 1. 获取应用的订阅信息  2. 让组件开始订阅

```go
func (a *DaprRuntime) beginPubSub(name string, ps pubsub.PubSub) error {
	var publishFunc func(ctx context.Context, msg *pubsubSubscribedMessage) error
    ......
	topicRoutes, err := a.getTopicRoutes()
    ......
}
```

## 获取应用订阅信息(AppCallback)

在 getTopicRoutes() 方法中，可以通过 HTTP 或者 gRPC 的方式来获取应用订阅信息： 

```go
func (a *DaprRuntime) getTopicRoutes() (map[string]TopicRoute, error) {
    ......
    if a.runtimeConfig.ApplicationProtocol == HTTPProtocol {
        // 走 http channel
		subscriptions, err = runtime_pubsub.GetSubscriptionsHTTP(a.appChannel, log)
	} else if a.runtimeConfig.ApplicationProtocol == GRPCProtocol {
        // 走 grpc channel
		client := runtimev1pb.NewAppCallbackClient(a.grpc.AppClient)
		subscriptions, err = runtime_pubsub.GetSubscriptionsGRPC(client, log)
	}
    ......
}
```

对于 HTTP 方式，调用的是 AppChannel 上定义的 InvokeMethod 方法，这个方法原来设计是用来实现 service invoke 的，dapr runtime 用来通过它将 service invoke 的 http inbound 请求转发给作为服务器端的应用。而在这里，被用来调用 `dapr/subscribe` 路径：

```go
func GetSubscriptionsHTTP(channel channel.AppChannel, log logger.Logger) ([]Subscription, error) {
    req := invokev1.NewInvokeMethodRequest("dapr/subscribe")
    channel.InvokeMethod(ctx, req)
    ......
}
```

> 感想：理论上说这也不是为一种方便的方式，只是总感觉有点怪怪，pubsub 模块的初始化用到了 service invoke 模块的功能。直接发个http请求代码也不复杂。另外 http AppChannel / app callback 的方法和 grpc  AppChannel / app callback 不对称，这在设计上缺乏美感。

对于 gRPC 方式，就比较老实的调用了 gRPC AppCallbackClient 的方法 ListTopicSubscriptions():

```go
resp, err = channel.ListTopicSubscriptions(context.Background(), &emptypb.Empty{})
```

## pubsub 组件开始订阅

在获取到应用的订阅信息之后，dapr runtime 就知道这个应用需要订阅哪些topic了。因此就可以继续开始订阅操作：

```go
func (a *DaprRuntime) beginPubSub(name string, ps pubsub.PubSub) error {
	var publishFunc func(ctx context.Context, msg *pubsubSubscribedMessage) error
    ......
    // 获取订阅信息
	topicRoutes, err := a.getTopicRoutes()
    ......
    // 开始订阅
    for topic, route := range v.routes {
        // 在当前 pubsub 组件上为每个 topic 进行订阅
        err := ps.Subscribe(pubsub.SubscribeRequest{
			Topic:    topic,
			Metadata: route.metadata,
        }, func(ctx context.Context, msg *pubsub.NewMessage) error {......}
    }
}
```

这里的 Subscribe() 方法的定义在 PubSub 接口上，每个 dapr pubsub 组件都会实现这个接口：

```go
type PubSub interface {
	Publish(req *PublishRequest) error
	Subscribe(req SubscribeRequest, handler Handler) error
}
```

handler 方法的具体实现后面再展开。

