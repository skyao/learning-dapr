---
type: docs
title: "客户端sdk为dapr提供订阅信息"
linkTitle: "SDK提供订阅信息"
weight: 30
date: 2022-04-24
description: >
  Dapr客户端sdk封装dapr api，接受dapr发出的ListTopicSubscriptions请求
---

## 工作原理

对于订阅信息而言，有四个关键的信息。在 dapr proto 中的定义如下：

```protobuf
message TopicSubscription {
  // Required. The name of the pubsub containing the topic below to subscribe to.
  string pubsub_name = 1;

  // Required. The name of topic which will be subscribed
  string topic = 2;

  // The optional properties used for this topic's subscription e.g. session id
  map<string,string> metadata = 3;

  // The optional routing rules to match against. In the gRPC interface, OnTopicEvent
  // is still invoked but the matching path is sent in the TopicEventRequest.
  TopicRoutes routes = 5;
}
```

pubsub_name 指定要使用的 pubsub component，topic 是要订阅的主题， metadata 携带扩展信息，而 routes 路由则是标记 dapr 应该如何将订阅到的事件发送给应用。

TODO：对于 HTTP 协议和 gRPC 协议处理会有不同。

java sdk中的封装如下：

```java
public class DaprTopicSubscription {
  private final String pubsubName;
  private final String topic;
  private final String route;
  private final Map<String, String> metadata;
}
```

dapr sdk 需要帮助应用方便的提供上述订阅信息。

## Java SDK 实现

在业务代码中使用 subscribe 功能的示例可参考文件 dapr java-sdk 中的代码 `/src/main/java/io/dapr/examples/pubsub/http/subscribe.java`，代码示意如下：

```java
// 启动应用，监听端口，一般喜欢使用 3000
public static void main(String[] args) throws Exception {
    ......
	DaprApplication.start(port); 
}

@RestController
public class SubscriberController {
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}")
  @PostMapping(path = "/testingtopic")
  public Mono<Void> handleMessage(@RequestBody(required = false) CloudEvent<String> cloudEvent) {
      ......
  }
}
```

### sdk收集订阅信息

上面代码中的 @Topic 注解是 dapr java sdk 提供的，用来标记需要进行 subscribe 的 topic，代码在`src/main/java/io/dapr/Topic.java`：

```java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Topic {
    String name();
    String pubsubName();
    String metadata() default "{}";
}
```

topic 的收集是典型的 springboot 风格，代码在 `sdk-springboot/src/main/java/io/dapr/springboot/DaprBeanPostProcessor.java`:

```java
@Component
public class DaprBeanPostProcessor implements BeanPostProcessor {
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    subscribeToTopics(bean.getClass(), embeddedValueResolver);
    return bean;
  }
}
```

subscribeToTopics() 方法通过扫描 @topic 注解和 @PostMapping 注解来获取订阅相关的信息：

```java
private static void subscribeToTopics(Class clazz, EmbeddedValueResolver embeddedValueResolver) {

    for (Method method : clazz.getDeclaredMethods()) {
      // 获取 @topic 注解
      Topic topic = method.getAnnotation(Topic.class);
      if (topic == null) {
        continue;
      }

      String route = topic.name();
      // 获取 @PostMapping 注解
      PostMapping mapping = method.getAnnotation(PostMapping.class);

      // 根据 PostMapping 注解获取 route 信息
      if (mapping != null && mapping.path() != null && mapping.path().length >= 1) {
        route = mapping.path()[0];
      } else if (mapping != null && mapping.value() != null && mapping.value().length >= 1) {
        route = mapping.value()[0];
      }

      String topicName = embeddedValueResolver.resolveStringValue(topic.name());
      String pubSubName = embeddedValueResolver.resolveStringValue(topic.pubsubName());
      if ((topicName != null) && (topicName.length() > 0) && pubSubName != null && pubSubName.length() > 0) {
        try {
          TypeReference<HashMap<String, String>> typeRef
                  = new TypeReference<HashMap<String, String>>() {};
          Map<String, String> metadata = MAPPER.readValue(topic.metadata(), typeRef);
          // 保存 subscribe 信息
          DaprRuntime.getInstance().addSubscribedTopic(pubSubName, topicName, route, metadata);
        } catch (JsonProcessingException e) {
          throw new IllegalArgumentException("Error while parsing metadata: " + e.toString());
        }
      }
    }
  }
```

DaprRuntime 是一个单例对象，这里保存有订阅的 topic 列表：

```java
class DaprRuntime {
    private final Set<String> subscribedTopics = new HashSet<>();
    private final List<DaprTopicSubscription> subscriptions = new ArrayList<>();
    
    public synchronized void addSubscribedTopic(String pubsubName,
                                                String topicName,
                                                String route,
                                                Map<String,String> metadata) {
        if (!this.subscribedTopics.contains(topicName)) {
            this.subscribedTopics.add(topicName);
            this.subscriptions.add(new DaprTopicSubscription(pubsubName, topicName, route, metadata));
        }
    }
}
```

### sdk暴露订阅信息

为了让 dapr 在 springboot 体系中方便使用，dapr java sdk 提供了 DaprController ，以提供诸如健康检查等通用功能，还有和dapr相关的各种端点，其中就有为 dapr runtime 提供订阅信息的接口：

```java
@RestController
public class DaprController {
  ......
  @GetMapping(path = "/dapr/subscribe", produces = MediaType.APPLICATION_JSON_VALUE)
  public byte[] daprSubscribe() throws IOException {
    return SERIALIZER.serialize(DaprRuntime.getInstance().listSubscribedTopics());
  }
}
```

通过这个URL，就可以将之前收集到的 topic 信息都暴露出去，可以在浏览器中直接访问 `http://127.0.0.1:3000/dapr/subscribe`，应答内容为:

```javascript
[{"pubsubName":"messagebus","topic":"testingtopic","route":"/testingtopic","metadata":{}}]
```



## Go sdk实现

在 go 业务代码中使用 subscribe 功能的示例可参考 https://github.com/dapr/go-sdk/blob/main/examples/pubsub/sub/sub.go，代码示意如下：

```go
func main() {
    s := daprd.NewService(":8080")
    err := s.AddTopicEventHandler(defaultSubscription, eventHandler)
    err = s.Start()
}

func eventHandler(ctx context.Context, e *common.TopicEvent) (retry bool, err error) {
	......
	return false, nil
}
```

### sdk收集订阅信息

Go sdk 中定义了 Service 接口

```go
// Service represents Dapr callback service.
type Service interface {
	// AddTopicEventHandler appends provided event handler with its topic and optional metadata to the service.
	// Note, retries are only considered when there is an error. Lack of error is considered as a success
	AddTopicEventHandler(sub *Subscription, fn TopicEventHandler) error
	......
}
```

Subscription 的定义如下：

```go
// Subscription represents single topic subscription.
type Subscription struct {
	PubsubName string `json:"pubsubname"`
	Topic string `json:"topic"`
	Metadata map[string]string `json:"metadata,omitempty"`
	Route string `json:"route"`
	......
}
```

这样订阅相关的主要4个参数就通过这个方式指明了。

### sdk暴露订阅信息

go sdk 中有 http 和 grpc 两套机制可以实现对外暴露访问端点。

http 的实现在 `http/topic.go` 中：

```go
func (s *Server) AddTopicEventHandler(sub *common.Subscription, fn common.TopicEventHandler) error {
	if err := s.topicRegistrar.AddSubscription(sub, fn); err != nil {
		return err
	}

    // 注册 http handle，关联 Route 和 fn
	s.mux.Handle(sub.Route, optionsHandler(http.HandlerFunc(
		func(w http.ResponseWriter, r *http.Request) {
            ......
            retry, err := fn(r.Context(), &te)
            ......
        }
    }
```

 grpc类似。

## 其他SDK

TODO

