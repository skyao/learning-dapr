---
title: "topic subscription"
linkTitle: "topic subscription"
weight: 100
date: 2022-07-24
description: >
  实现 pub/sub 中的 topic 订阅
---

## 读取 topic 订阅注解

订阅 topic 的具体代码实现在类 DaprBeanPostProcessor 的 subscribeToTopics() 方法中，在 bean 初始化时被调用。

topic 注解使用的例子如下：

```java
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}",
          rule = @Rule(match = "event.type == \"v2\"", priority = 1))
  @PostMapping(path = "/testingtopicV2")
  public Mono<Void> handleMessageV2(@RequestBody(required = false) CloudEvent cloudEvent) {
    ......
  }
```

### 读取 topic 注解

现在需要在 postProcessBeforeInitialization() 方法中扫描并解析所有有 topic 注解的 bean：

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  subscribeToTopics(bean.getClass(), embeddedValueResolver);
  return bean;
}

private static void subscribeToTopics(Class clazz, EmbeddedValueResolver embeddedValueResolver) {
    if (clazz == null) {
      return;
    }

    // 先用 Superclass 做一次递归调用，这样就会从当前类的父类开始先推衍
    // 由于每次都是父类先执行，因此这会一直递归到最顶层的 Object 类 
    subscribeToTopics(clazz.getSuperclass(), embeddedValueResolver);
    // 取当前类的所有方法
    for (Method method : clazz.getDeclaredMethods()) {
      // 然后看方法上是不是标记了 dapr 的 topic 注解
      Topic topic = method.getAnnotation(Topic.class);
      if (topic == null) {
        continue;
      }

      // 如果方法上有标记 dapr 的 topic 注解，则开始处理
      // 先获取 topic 注解上的属性 topic name, pubsub name, rule 
      Rule rule = topic.rule();
      String topicName = embeddedValueResolver.resolveStringValue(topic.name());
      String pubSubName = embeddedValueResolver.resolveStringValue(topic.pubsubName());
      // rule 也是一个注解，获取 match 属性
      String match = embeddedValueResolver.resolveStringValue(rule.match());
      if ((topicName != null) && (topicName.length() > 0) && pubSubName != null && pubSubName.length() > 0) {
        // topicName 和 pubSubName 不能为空 （metadata 可以为空，rule可以为空）
        try {
          TypeReference<HashMap<String, String>> typeRef
                  = new TypeReference<HashMap<String, String>>() {};
          // 读取 topic 注解上的 metadata 属性
          Map<String, String> metadata = MAPPER.readValue(topic.metadata(), typeRef);
          // 读取路由信息，细节看下一节
          List<String> routes = getAllCompleteRoutesForPost(clazz, method, topicName);
          for (String route : routes) {
            // 将读取的路由信息添加到 dapr runtime 中。
            // 细节看下一节
            DaprRuntime.getInstance().addSubscribedTopic(
                pubSubName, topicName, match, rule.priority(), route, metadata);
          }
        } catch (JsonProcessingException e) {
          throw new IllegalArgumentException("Error while parsing metadata: " + e);
        }
      }
    }
  }
```

### 读取路由信息

路由信息配置方法如下：

```java
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}",
          rule = @Rule(match = "event.type == \"v2\"", priority = 1))
  @PostMapping(path = "/testingtopicV2")
  public Mono<Void> handleMessageV2(@RequestBody(required = false) CloudEvent cloudEvent) {
    ......
  }
```

getAllCompleteRoutesForPost() 方法负责读取 @rule 注解相关的路由信息：

```java
private static List<String> getAllCompleteRoutesForPost(Class clazz, Method method, String topicName) {
    List<String> routesList = new ArrayList<>();
    RequestMapping clazzRequestMapping =
        (RequestMapping) clazz.getAnnotation(RequestMapping.class);
    String[] clazzLevelRoute = null;
    if (clazzRequestMapping != null) {
      clazzLevelRoute = clazzRequestMapping.value();
    }
    // 读取该方法上的路由信息，注意必须是 POST
    String[] postValueArray = getRoutesForPost(method, topicName);
    if (postValueArray != null && postValueArray.length >= 1) {
      for (String postValue : postValueArray) {
        if (clazzLevelRoute != null && clazzLevelRoute.length >= 1) {
          for (String clazzLevelValue : clazzLevelRoute) {
            // 完整的路由路径应该是类级别 + 方法级别
            String route = clazzLevelValue + confirmLeadingSlash(postValue);
            routesList.add(route);
          }
        } else {
          routesList.add(postValue);
        }
      }
    }
    return routesList;
  }
```

getRoutesForPost() 方法用来读取 @topic 注解所在方法的 @PostMapping 注解，以便获得路由的 path 信息，对应例子如下：

```java
  @Topic(name = "testingtopic", pubsubName = "${myAppProperty:messagebus}",
          rule = @Rule(match = "event.type == \"v2\"", priority = 1))
  @PostMapping(path = "/testingtopicV2")
  public Mono<Void> handleMessageV2(@RequestBody(required = false) CloudEvent cloudEvent) {
    ......
  }
```

getRoutesForPost() 方法的代码实现如下：

```java
private static String[] getRoutesForPost(Method method, String topicName) {
    String[] postValueArray = new String[] {topicName};
    // 读取 PostMapping 注解
    PostMapping postMapping = method.getAnnotation(PostMapping.class);
    if (postMapping != null) {
      // 如果有 PostMapping 注解
      if (postMapping.path() != null && postMapping.path().length >= 1) {
        // 如果 path 属性有设置则从 path 属性取值
        postValueArray = postMapping.path();
      } else if (postMapping.value() != null && postMapping.value().length >= 1) {
        // 如果 path 属性没有设置则直接从 PostMapping 注解的 value 中取值
        postValueArray = postMapping.value();
      }
    } else {
      // 如果没有 PostMapping 注解，则尝试读取 RequestMapping 注解
      RequestMapping reqMapping = method.getAnnotation(RequestMapping.class);
      for (RequestMethod reqMethod : reqMapping.method()) {
        // 要求 RequestMethod 为 POST
        if (reqMethod == RequestMethod.POST) {
          // 同样读取 path 或者 value 的值
          if (reqMapping.path() != null && reqMapping.path().length >= 1) {
            postValueArray = reqMapping.path();
          } else if (reqMapping.value() != null && reqMapping.value().length >= 1) {
            postValueArray = reqMapping.value();
          }
          break;
        }
      }
    }
    return postValueArray;
  }
```

getRoutesForPost() 方法的解读，就是从标记了 @topic 注解的方法上读取路由信息，也就是后续订阅的事件应该发送的地址。读取的逻辑为：

1. 优先读取 PostMapping 注解，没有的话读取 RequestMethod 为 POST 的 RequestMapping 注解
2. 优先读取上述注解的 path 属性，没有的话读取 value

## 保存 topic 订阅信息

topic 订阅信息在读取之后，就会通过 DaprRuntime 的 addSubscribedTopic() 方法保存起来：

```java
public synchronized void addSubscribedTopic(String pubsubName,
                                              String topicName,
                                              String match,
                                              int priority,
                                              String route,
                                              Map<String,String> metadata) {
    // 用 pubsubName 和 topicName 做 key
    DaprTopicKey topicKey = new DaprTopicKey(pubsubName, topicName);

    // 获取 key 对应的 builder，没有的话就创建一个
    DaprSubscriptionBuilder builder = subscriptionBuilders.get(topicKey);
    if (builder == null) {
      builder = new DaprSubscriptionBuilder(pubsubName, topicName);
      subscriptionBuilders.put(topicKey, builder);
    }

    // match 不为空则添加 rule，为空则采用默认路径
    if (match.length() > 0) {
      builder.addRule(route, match, priority);
    } else {
      builder.setDefaultPath(route);
    }

    if (metadata != null && !metadata.isEmpty()) {
      builder.setMetadata(metadata);
    }
  }
```

考虑到调用的地方代码是：

```java
// 读取路由信息
List<String> routes = getAllCompleteRoutesForPost(clazz, method, topicName);
for (String route : routes) {
  // 将读取的路由信息添加到 dapr runtime 中。
  DaprRuntime.getInstance().addSubscribedTopic(
      pubSubName, topicName, match, rule.priority(), route, metadata);
}
```

所以前面的读取流程可以理解为就是读取和 topic 订阅有关的上述6个参数，然后保存起老。

## 应答 topic 订阅信息

在 DaprController 中，daprSubscribe() 方法对外暴露路径 `/dapr/subscribe` ，以便让 dapr sidecar 可以通过读取该路径来获取当前应用的 topic 订阅信息：

```java
@GetMapping(path = "/dapr/subscribe", produces = MediaType.APPLICATION_JSON_VALUE)
public byte[] daprSubscribe() throws IOException {
  return SERIALIZER.serialize(DaprRuntime.getInstance().listSubscribedTopics());
}
```

而 DaprRuntime 的 listSubscribedTopics() 方法获取的就是前面保存起来的 topic 订阅信息：

```java
  public synchronized DaprTopicSubscription[] listSubscribedTopics() {
    List<DaprTopicSubscription> values = subscriptionBuilders.values().stream()
            .map(b -> b.build()).collect(Collectors.toList());
    return values.toArray(new DaprTopicSubscription[0]);
  }
```

## 流程总结

整个 topic 订阅流程的示意图如下：

```plantuml
title topic subscription
hide footbox
skinparam style strictuml


box "Application" #LightBlue
participant DaprBeanPostProcessor
participant bean
participant DaprRuntime
participant DaprController
end box
participant daprd

-> DaprBeanPostProcessor: postProcessBeforeInitialization(bean)

DaprBeanPostProcessor -> bean: get @topic
bean --> DaprBeanPostProcessor
 
alt if bean has @topic
DaprBeanPostProcessor -> bean: parse @topic @rule
bean --> DaprBeanPostProcessor: pubsub name, topic name, match,\n priority, routes, metadata

DaprBeanPostProcessor -> DaprRuntime: addSubscribedTopic()

DaprRuntime -> DaprRuntime: save in map\n subscriptionBuilders

DaprRuntime --> DaprBeanPostProcessor
end
<-- DaprBeanPostProcessor

daprd -> DaprController: get subscription
DaprController -> DaprRuntime: listSubscribedTopics()
DaprRuntime --> DaprController
DaprController --> daprd
```


