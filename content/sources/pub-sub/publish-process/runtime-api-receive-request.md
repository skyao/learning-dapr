---
title: "Dapr Runtime 处理来自客户端的 publish 请求"
linkTitle: "Runtime处理publish请求"
weight: 50
date: 2022-04-21
description: >
  Dapr Runtime 接收来自客户端的 publish 请求的代码分析
---

在 dapr runtime 中，提供 HTTP 和 gRPC 两种协议，前面 runtime 初始化时介绍了 HTTP 和 gRPC 两种协议是如何在 runtime 初始化时准备好接收来自客户端的 publish 请求的。现在我们介绍在接收到来自客户端的 publish 请求后，dapr runtime 是如何处理请求的。

## gRPC API

在 gRPC API 的实现中，PublishEvent() 方法负责处理接收到的 publish 请求，其主要流程大体是如下4个步骤：

```go
type api struct {
    pubsubAdapter              runtimePubsub.Adapter
}

func (a *api) PublishEvent(ctx context.Context, in *runtimev1pb.PublishEventRequest) (*emptypb.Empty, error) {
  // 1. 根据名称找到可以处理请求的 pubsub 组件
  thepubsub := a.pubsubAdapter.GetPubSub(pubsubName)
  // 2. 处理参数的细节：如是否要封装为 cloudevent
  // 细节忽略，后续展开
  // 3. 构建 PublishRequest 请求对象
  req := pubsub.PublishRequest{
		PubsubName: pubsubName,
		Topic:      topic,
		Data:       data,
		Metadata:   in.Metadata,
	}
  // 4. 未退 pubsub 组件来负责具体的请求发送
  err := a.pubsubAdapter.Publish(&req)
}
```

### 查找处理请求的 pubsub 组件

```go
  // 检查是否有初始化 pubsubAdapter，没有的话报错退出
  if a.pubsubAdapter == nil {
		err := status.Error(codes.FailedPrecondition, messages.ErrPubsubNotConfigured)
		apiServerLogger.Debug(err)
		return &emptypb.Empty{}, err
	}

	pubsubName := in.PubsubName
  // 检查请求，pubsubName 参数不能为空
	if pubsubName == "" {
		err := status.Error(codes.InvalidArgument, messages.ErrPubsubEmpty)
		apiServerLogger.Debug(err)
		return &emptypb.Empty{}, err
	}

  // 根据 pubsubName 参数在 pubsubAdapter 中找到对应的组件
	thepubsub := a.pubsubAdapter.GetPubSub(pubsubName)
	if thepubsub == nil {
    // 如果找不到，则报错退出
		err := status.Errorf(codes.InvalidArgument, messages.ErrPubsubNotFound, pubsubName)
		apiServerLogger.Debug(err)
		return &emptypb.Empty{}, err
	}
```

GetPubSub() 方法的实现很简单，就是根据 pubsubName 在现有已经初始化的 pubsub 组件中进行简单的map查找：

```go
// GetPubSub is an adapter method to find a pubsub by name.
func (a *DaprRuntime) GetPubSub(pubsubName string) pubsub.PubSub {
	ps, ok := a.pubSubs[pubsubName]
	if !ok {
		return nil
	}
	return ps.component
}
```

### 委托 pubsub 组件发送请求

```go
func (a *DaprRuntime) Publish(req *pubsub.PublishRequest) error {
  // 这里又根据名称做了一次查找
  // TBD：可以考虑做代码优化了，从前面把找到的组件传递过来就好了
	ps, ok := a.pubSubs[req.PubsubName]
	if !ok {
		return runtimePubsub.NotFoundError{PubsubName: req.PubsubName}
	}

  // 检查 pubsub 操作是否被容许
	if allowed := a.isPubSubOperationAllowed(req.PubsubName, req.Topic, ps.scopedPublishings); !allowed {
		return runtimePubsub.NotAllowedError{Topic: req.Topic, ID: a.runtimeConfig.ID}
	}

  // 执行策略
	policy := a.resiliency.ComponentOutboundPolicy(a.ctx, req.PubsubName)
	return policy(func(ctx context.Context) (err error) {
    // 最终调用到底层实际组件的 Publish 方法来发送请求
		return ps.component.Publish(req)
	})
}
```



## HTTP API

HTTP API 的处理方式和 gRPC API 是一致的，只是 HTTP API 这边由于 HTTP 协议的原因，在请求参数的获取上无法像 gRPC API 那样有一个的 runtimev1pb.PublishEventRequest 对象可以完整的封装所有请求参数，HTTP API 会多出一个请求参数的获取过程。

### 从 HTTP 请求中获取所有参数

HTTP API 实现中的 onPublish() 方法的前面一段代码就是在处理如何从 HTTP 请求中获取 publish 所需的所有参数：

```go
func (a *api) onPublish(reqCtx *fasthttp.RequestCtx) {
  // 1. pubsubName
  pubsubName := reqCtx.UserValue(pubsubnameparam).(string)
  // 2. topic
  topic := reqCtx.UserValue(topicParam).(string)
  // 3. data
  body := reqCtx.PostBody()
  // 4. data content type
	contentType := string(reqCtx.Request.Header.Peek("Content-Type"))
  // 5. metadata
	metadata := getMetadataFromRequest(reqCtx)
  
  // 后续处理和 gRPC 协议一致
  ......
}
```

