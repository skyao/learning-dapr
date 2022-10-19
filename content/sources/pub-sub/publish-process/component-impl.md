---
title: "组件实现"
linkTitle: "组件的publish实现"
weight: 60
date: 2022-04-21
description: >
  组件实现publish的实际功能
---



## 组件接口中的 Publish() 方法定义

在 dapr runtime API 实现（包括 HTTP API 和 gRPC API）和底层 pubsub 组件之间，还有一个简单的内部接口，定义了 pubsub 组件的功能：

```go
// PubSub is the interface for message buses.
type PubSub interface {
	Init(metadata Metadata) error
	Features() []Feature
	Publish(req *PublishRequest) error
	Subscribe(ctx context.Context, req SubscribeRequest, handler Handler) error
	Close() error
}
```

其中的 Publish() 用来发送消息。请求参数 PublishRequest 的字段和 Dapr API 定义中保持一致：

```go
// PublishRequest is the request to publish a message.
type PublishRequest struct {
	Data        []byte            `json:"data"`
	PubsubName  string            `json:"pubsubname"`
	Topic       string            `json:"topic"`
	Metadata    map[string]string `json:"metadata"`
	ContentType *string           `json:"contentType,omitempty"`
}
```



## redis 组件实现

以 redis stream 为例，看看 publish 方法的实现：

```go
func (r *redisStreams) Publish(req *pubsub.PublishRequest) error {
	_, err := r.client.XAdd(r.ctx, &redis.XAddArgs{
		Stream:       req.Topic,
		MaxLenApprox: r.metadata.maxLenApprox,
		Values:       map[string]interface{}{"data": req.Data},
	}).Result()
	if err != nil {
		return fmt.Errorf("redis streams: error from publish: %s", err)
	}

	return nil
}
```

redis stream 的实现很简单，req.Topic 参数指定要写入的 redis stream，内容为一个map，其中 key "data" 的值为 req.Data。







