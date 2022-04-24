---
type: docs
title: "发布相关的Runtime初始化"
linkTitle: "Runtime初始化"
weight: 20
date: 2022-04-21
description: >
  Dapr Runtime中和发布相关的初始化流程
---

在 dapr runtime 启动进行初始化时，需要开启 API 端口并挂载相应的 handler 来接收并处理发布订阅中的发布请求。另外需要根据配置文件启动 pubsub component 以便连接到外部 message broker。

## Dapr HTTP API Server(outbound)

### 在 dapr runtime 中启动 HTTP server

类似 service invoke。

在 dapr runtime 启动时的初始化过程中，会启动 HTTP server， 代码在 `pkg/runtime/runtime.go` 中。

### 挂载 PubSub 的 HTTP  端点

在 HTTP API 的初始化过程中，会在 fast http server 上挂载 PubSub 的 HTTP  端点，代码在 `pkg/http/api.go` 中：

```go
func NewAPI(
  appID string,
	appChannel channel.AppChannel,
	directMessaging messaging.DirectMessaging,
  ......
  	shutdown func()) API {
  
  	api := &api{
		appChannel:               appChannel,
		directMessaging:          directMessaging,
		......
	}
  
  	// 附加 PubSub 的 HTTP 端点
  	api.endpoints = append(api.endpoints, api.constructPubSubEndpoints()...)
}
```

PubSub 的 HTTP 端点的具体信息在 constructPubSubEndpoints() 方法中：

```go
func (a *api) constructPubSubEndpoints() []Endpoint {
	return []Endpoint{
		{
			Methods: []string{fasthttp.MethodPost, fasthttp.MethodPut},
			Route:   "publish/{pubsubname}/{topic:*}",
			Version: apiVersionV1,
			Handler: a.onPublish,
		},
	}
}
```

注意这里的 Route 路径 "publish/{pubsubname}/{topic:*}"， dapr sdk 就是就通过这样的 url 来发起 HTTP publish 请求。

```plantuml
title Dapr Publish HTTP API 
hide footbox
skinparam style strictuml

participant daprd_client [
    =daprd
    ----
    producer
]

-[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500\n/v1.0/publish/{pubsubname}/{topic:*}
|||
<[#blue]-- daprd_client
```

## Dapr gRPC API Server(outbound)

### 启动 gRPC 服务器

类似 service invoke。

在 dapr runtime 启动时的初始化过程中，会启动 gRPC server， 代码在 `pkg/runtime/runtime.go` 中。

### 注册 Dapr API

为了让 dapr runtime 的 gRPC  服务器能挂载 Dapr API，需要进行注册上去。

注册的代码实现在 `pkg/grpc/server.go` 中， StartNonBlocking() 方法在启动 grpc 服务器时，会进行服务注册。

### Dapr_ServiceDesc 定义

在文件 `pkg/proto/runtime/v1/dapr_grpc.pb.go` 中有 Dapr Service 的 grpc 服务定义，这是 protoc 生成的 gRPC 代码。

Dapr_ServiceDesc 中有 Dapr Service 各个方法的定义，和发布相关的是 `InvokeService` 方法：

```go
var Dapr_ServiceDesc = grpc.ServiceDesc{
	ServiceName: "dapr.proto.runtime.v1.Dapr",
	HandlerType: (*DaprServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "PublishEvent",				  # 注册方法名
			Handler:    _Dapr_PublishEvent_Handler,	  # 关联实现的 Handler
		},
        ......
        },
	},
	Metadata: "dapr/proto/runtime/v1/dapr.proto",
}
```

这一段是告诉 gRPC server： 如果收到访问 `dapr.proto.runtime.v1.Dapr` 服务的 `PublishEvent` 方法的  gRPC 请求，请把请求转给 `_Dapr_PublishEvent_Handler` 处理。

```plantuml
title Dapr publish gRPC API 
hide footbox
skinparam style strictuml

participant daprd_client [
    =daprd
    ----
    producer
]

-[#blue]> daprd_client : gRPC (localhost)
note right: gRPC API @ 50001\n/dapr.proto.runtime.v1.Dapr/PublishEvent
|||
<[#blue]-- daprd_client
```

而 `PublishEvent` 方法相关联的 handler 方法 `_Dapr_PublishEvent_Handler `的实现代码是：

```go
func _Dapr_PublishEvent_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(PublishEventRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(DaprServer).PublishEvent(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/dapr.proto.runtime.v1.Dapr/PublishEvent",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(DaprServer).PublishEvent(ctx, req.(*PublishEventRequest))
	}
	return interceptor(ctx, in, info, handler)
}
```

最后调用到了 DaprServer 接口实现的 PublishEvent 方法，也就是 gPRC API 实现。

## pubsub 组件初始化

为了提供对 pubsub 的功能支持，需要为 dapr runtime 配置 pubsub component。

### pubSubRegistry 和 pubSubs 列表

DaprRuntime 的结构体中保存有 pubSubRegistry 和 pubSubs 列表：

```go
type DaprRuntime struct {
	......
	pubSubRegistry         pubsub_loader.Registry
	pubSubs                map[string]pubsub.PubSub
	......
}
```

runtime 构建时会初始化这两个结构体：

```go
func NewDaprRuntime(runtimeConfig *Config, globalConfig *config.Configuration, accessControlList *config.AccessControlList, resiliencyProvider resiliency.Provider) *DaprRuntime {
	ctx, cancel := context.WithCancel(context.Background())
	return &DaprRuntime{
		......
		pubSubs:                map[string]pubsub.PubSub{},
		pubSubRegistry:         pubsub_loader.NewRegistry(),
        ......
```

### PubSubRegistry 保存pubsub组件列表

pubSubRegistry 用于保存 dapr runtime 中支持的所有 pubsub component ：

```go
pubSubRegistry struct {
    messageBuses map[string]func() pubsub.PubSub
}
```

在 runtime binary （`cmd/daprd/main.go`）的代码中，会列举出所有的 pubsub component ，这也是 darp 和 conponents-contrib 两个仓库的直接联系：

```go
err = rt.Run(
		......
		runtime.WithPubSubs(
			pubsub_loader.New("azure.eventhubs", func() pubs.PubSub {
				return pubsub_eventhubs.NewAzureEventHubs(logContrib)
			}),
			pubsub_loader.New("azure.servicebus", func() pubs.PubSub {
				return servicebus.NewAzureServiceBus(logContrib)
			}),
			pubsub_loader.New("gcp.pubsub", func() pubs.PubSub {
				return pubsub_gcp.NewGCPPubSub(logContrib)
			}),
			pubsub_loader.New("hazelcast", func() pubs.PubSub {
				return pubsub_hazelcast.NewHazelcastPubSub(logContrib)
			}),
			pubsub_loader.New("jetstream", func() pubs.PubSub {
				return pubsub_jetstream.NewJetStream(logContrib)
			}),
			pubsub_loader.New("kafka", func() pubs.PubSub {
				return pubsub_kafka.NewKafka(logContrib)
			}),
			pubsub_loader.New("mqtt", func() pubs.PubSub {
				return pubsub_mqtt.NewMQTTPubSub(logContrib)
			}),
			pubsub_loader.New("natsstreaming", func() pubs.PubSub {
				return natsstreaming.NewNATSStreamingPubSub(logContrib)
			}),
			pubsub_loader.New("pulsar", func() pubs.PubSub {
				return pubsub_pulsar.NewPulsar(logContrib)
			}),
			pubsub_loader.New("rabbitmq", func() pubs.PubSub {
				return rabbitmq.NewRabbitMQ(logContrib)
			}),
			pubsub_loader.New("redis", func() pubs.PubSub {
				return pubsub_redis.NewRedisStreams(logContrib)
			}),
			pubsub_loader.New("snssqs", func() pubs.PubSub {
				return pubsub_snssqs.NewSnsSqs(logContrib)
			}),
			pubsub_loader.New("in-memory", func() pubs.PubSub {
				return pubsub_inmemory.New(logContrib)
			}),
		),
    ......
)
```

runtime 在初始化时会将这些 pubsub component 信息保存在 pubSubRegistry 中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    ......
	a.pubSubRegistry.Register(opts.pubsubs...)
}
```

需要主要的是，pubSubRegistry 中保存的组件列表是所有的被dapr runtime 支持的组件列表，但是，不是每个组件在 runtime 启动时都会被装载。组件的安装时按需的，由组件配置文件（yaml）来决定装载和初始化那些组件的示例。

### runtime 装载 pubsub 组件

组件在 dapr runtime 初始化时统一装载：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
    ......
	a.pubSubRegistry.Register(opts.pubsubs...)
	a.secretStoresRegistry.Register(opts.secretStores...)
	a.stateStoreRegistry.Register(opts.states...)
    ......
    err = a.loadComponents(opts)
    a.flushOutstandingComponents()
    ......
}
```

有两种实现，KubernetesMode 和 StandaloneMode：

```go
func (a *DaprRuntime) loadComponents(opts *runtimeOpts) error {
    	var loader components.ComponentLoader

	switch a.runtimeConfig.Mode {
	case modes.KubernetesMode:
		loader = components.NewKubernetesComponents(a.runtimeConfig.Kubernetes, a.namespace, a.operatorClient, a.podName)
	case modes.StandaloneMode:
		loader = components.NewStandaloneComponents(a.runtimeConfig.Standalone)
	default:
		return errors.Errorf("components loader for mode %s not found", a.runtimeConfig.Mode)
	}
    comps, err := loader.LoadComponents()
    ......
}
```

KubernetesMode 下读取的是 k8s 下的 component CRD：

```go
func (k *KubernetesComponents) LoadComponents() ([]components_v1alpha1.Component, error) {
	resp, err := k.client.ListComponents(context.Background(), &operatorv1pb.ListComponentsRequest{
		Namespace: k.namespace,
		PodName:   k.podName,
	}, ......
}
```

StandaloneMode 下读取的是由  ComponentsPath 配置(`--componentspath`)指定的目录下的 component CRD 文件：

```go
func (s *StandaloneComponents) LoadComponents() ([]components_v1alpha1.Component, error) {
	files, err := os.ReadDir(s.config.ComponentsPath)
	......
}
```

## 总结

在完成 HTTP server 和 gRPC server 的初始化之后，dapr runtime 就做好了接收 publish 请求的准备。



