---
title: "测试源代码"
linkTitle: "测试源代码"
weight: 100
date: 2022-04-26
description: >
  Dapr的kafka pubsub component认证测试的源代码
---

源代码文件只有一个 `kafka_test.go`

## 准备工作

### kafka component

创建 kafka pubsub component 的代码：

```go
	log := logger.NewLogger("dapr.components")
	component := pubsub_loader.New("kafka", func() pubsub.PubSub {
		return pubsub_kafka.NewKafka(log)
	})
```

直接创建一个  kafka pubsub component 的实例。

### consumer group

```go
	// For Kafka, we should ensure messages are received in order.
	consumerGroup1 := watcher.NewOrdered()
	// This watcher is across multiple consumers in the same group
	// so exact ordering is not expected.
	consumerGroup2 := watcher.NewUnordered()
```

consumerGroup1 是有序的，consumerGroup2因为是在同一个group中跨了多个consumer因此只是无序的。

### application函数

application 用来消费从topic中得到的消息：

```go
	// Application logic that tracks messages from a topic.
	application := func(messages *watcher.Watcher) app.SetupFn {
		return func(ctx flow.Context, s common.Service) error {
			// Simulate periodic errors.
			sim := simulate.PeriodicError(ctx, 100)

			// Setup the /orders event handler.
			return multierr.Combine(
				s.AddTopicEventHandler(&common.Subscription{
					PubsubName: "messagebus",
					Topic:      "neworder",
					Route:      "/orders",
				}, func(_ context.Context, e *common.TopicEvent) (retry bool, err error) {
					if err := sim(); err != nil {
						return true, err
					}

					// Track/Observe the data of the event.
					messages.Observe(e.Data)
					return false, nil
				}),
			)
		}
	}
```

其中 application 订阅了名为 messagebus 的 pubsub 组件的 neworder topic，对应出来接收消息的地址为 "/orders"。

application 在接收到消息时，会讲消息保存到 Watcher 中以供后面检查。

另外 application 会模拟偶尔出错的情况，一遍进行retry的测试。



### sendRecvTest函数

sendRecvTest函数用于测试发送消息到topic的逻辑，并验证应用是否接收到消息：

```go
const (
  numMessages       = 1000
  ）

// Test logic that sends messages to a topic and
	// verifies the application has received them.
	sendRecvTest := func(metadata map[string]string, messages ...*watcher.Watcher) flow.Runnable {
		_, hasKey := metadata[messageKey]
		return func(ctx flow.Context) error {
      // 获取client，里面封装还有dapr go sdk的 Client interface
			client := sidecar.GetClient(ctx, sidecarName1)

			// Declare what is expected BEFORE performing any steps
			// that will satisfy the test.
			msgs := make([]string, numMessages)
			for i := range msgs {
				msgs[i] = fmt.Sprintf("Hello, Messages %03d", i)
			}
			for _, m := range messages {
				m.ExpectStrings(msgs...)
			}
			// If no key it provided, create a random one.
			// For Kafka, this will spread messages across
			// the topic's partitions.
			if !hasKey {
				metadata[messageKey] = uuid.NewString()
			}

			// Send events that the application above will observe.
			ctx.Log("Sending messages!")
			for _, msg := range msgs {
				ctx.Logf("Sending: %q", msg)
        // 通过 dapr client 的 PublishEvent 方法发送消息到topic
        // 对应的 pubsubName 为 "messagebus"，topicName 为 "neworder"
        // 消息内容为前面准备的 "Hello, Messages %03d"
				err := client.PublishEvent(
					ctx, pubsubName, topicName, msg,
					dapr.PublishEventWithMetadata(metadata))
				require.NoError(ctx, err, "error publishing message")
			}

			// Do the messages we observed match what we expect?
			for _, m := range messages {
        // 一分钟内应该能收到全部消息
				m.Assert(ctx, time.Minute)
			}

			return nil
		}
	}
```

### sendMessagesInBackground函数



### assertMessages函数



## 测试流程



### 准备步骤

#### 启动Docker Compose

```go
clusterName       = "kafkacertification"
dockerComposeYAML = "docker-compose.yml"		

// Run Kafka using Docker Compose.
Step(dockercompose.Run(clusterName, dockerComposeYAML)).
```

#### 等待broker启动

```go
var (
	brokers          = []string{"localhost:19092", "localhost:29092", "localhost:39092"}
)
Step("wait for broker sockets",
			network.WaitForAddresses(5*time.Minute, brokers...)).
```

等待 broker 启动完成打开端口。

> 这是 consumer1 对应的 kafka 集群，不需要认证

#### 等待kafka集群准备完成

```go
		Step("wait for kafka readiness", retry.Do(time.Second, 30, func(ctx flow.Context) error {
			config := sarama.NewConfig()
			config.ClientID = "test-consumer"
			config.Consumer.Return.Errors = true

			// Create new consumer
			client, err := sarama.NewConsumer(brokers, config)
			if err != nil {
				return err
			}
			defer client.Close()

			// Ensure the brokers are ready by attempting to consume
			// a topic partition.
			_, err = client.ConsumePartition("myTopic", 0, sarama.OffsetOldest)

			return err
		})).
```

#### 等待oauth认证完成

```go
		Step("wait for Dapr OAuth client", retry.Do(10*time.Second, 6, func(ctx flow.Context) error {
			httpClient := &http.Client{
				Transport: &http.Transport{
					TLSClientConfig: &tls.Config{
						InsecureSkipVerify: true, // test server certificate is not trusted.
					},
				},
			}

			resp, err := httpClient.Get(oauthClientQuery)
			if err != nil {
				return err
			}
			if resp.StatusCode != 200 {
				return fmt.Errorf("oauth client query for 'dapr' not successful")
			}
			return nil
		})).
```



### 启动应用和sidecar

#### 启动app-1

准备 app-1 以接收有序消息：

```go
const (
	appID1            = "app-1"
  appPort           = 8000
)
consumerGroup1 := watcher.NewOrdered()

// Run the application logic above.
Step(app.Run(appID1, fmt.Sprintf(":%d", appPort),
             application(consumerGroup1))).
```

运行应用"app-1"，端口 8000，使用 consumerGroup1 作为watcher，要求收到的消息是有序的。

然后启动 app-1 对应的带 kafka pubsub 组件的 sidecar：

```go
const (
	sidecarName1      = "dapr-1"
)

Step(sidecar.Run(sidecarName1,
			embedded.WithComponentsPath("./components/consumer1"),
			embedded.WithAppProtocol(runtime.HTTPProtocol, appPort),
			embedded.WithDaprGRPCPort(runtime.DefaultDaprAPIGRPCPort),
			embedded.WithDaprHTTPPort(runtime.DefaultDaprHTTPPort),
			runtime.WithPubSubs(component))).
```

组件配置文件路径为 "./components/consumer1"，里面的组件配置为无认证，连接的broker为 "localhost:19092,localhost:29092,localhost:39092", consumer group 为 kafkaCertification1.

> 备注：这是直接在当前 go test 里面启动了一个dapr runtime，并注入了一个 kafka pubsub component，够狠



#### 启动app-2

运行应用"app-2"，使用 consumerGroup2 作为watcher，要求收到的消息可以是无序的。

```go
const (
	appID2            = "app-2"
)
		// Run the second application.
		Step(app.Run(appID2, fmt.Sprintf(":%d", appPort+portOffset),
			application(consumerGroup2))).
```

同样启动 app-2 对应的带 kafka pubsub 组件的 sidecar：

```go
const (
	sidecarName2      = "dapr-2"
)
		// Run the Dapr sidecar with the Kafka component.
		Step(sidecar.Run(sidecarName2,
			embedded.WithComponentsPath("./components/mtls-consumer"),
			embedded.WithAppProtocol(runtime.HTTPProtocol, appPort+portOffset),
			embedded.WithDaprGRPCPort(runtime.DefaultDaprAPIGRPCPort+portOffset),
			embedded.WithDaprHTTPPort(runtime.DefaultDaprHTTPPort+portOffset),
			embedded.WithProfilePort(runtime.DefaultProfilePort+portOffset),
			runtime.WithPubSubs(component))).
```

组件配置文件路径为 "./components/mtls-consumer"，里面的组件配置为 mtls 认证，连接的broker为 "localhost:19094,localhost:29094,localhost:39094", consumer group 为 kafkaCertification2.

#### 启动app-3

运行应用"app-3"，使用 consumerGroup2 作为watcher，要求收到的消息可以是无序的。

```go
const (
	appID3            = "app-3"
)
		// Run the third application.
		Step(app.Run(appID3, fmt.Sprintf(":%d", appPort+portOffset*2),
			application(consumerGroup2))).
```

同样启动 app-3 对应的带 kafka pubsub 组件的 sidecar：

```go
const (
	sidecarName3      = "dapr-3"
)
		// Run the Dapr sidecar with the Kafka component.
		Step(sidecar.Run(sidecarName3,
			embedded.WithComponentsPath("./components/oauth-consumer"),
			embedded.WithAppProtocol(runtime.HTTPProtocol, appPort+portOffset*2),
			embedded.WithDaprGRPCPort(runtime.DefaultDaprAPIGRPCPort+portOffset*2),
			embedded.WithDaprHTTPPort(runtime.DefaultDaprHTTPPort+portOffset*2),
			embedded.WithProfilePort(runtime.DefaultProfilePort+portOffset*2),
			runtime.WithPubSubs(component))).
```

组件配置文件路径为 "./components/oauth-consumer"，里面的组件配置为 oauth 认证，连接的broker为 "localhost:19093,localhost:29093,localhost:39093", consumer group 为 kafkaCertification2.

启动 app-3 时，重置 consumerGroup2：

```go
Step("reset", flow.Reset(consumerGroup2)).
```



### 运行简单有序测试

由于使用了同一个 "partitionKey"，因此会被发送到kafka topic 的同一个分区，这样消息接收的顺序就会和发送的顺序保持一致。

```go
const（	
messageKey        = "partitionKey"	
）
// Set the partition key on all messages so they
	// are written to the same partition.
	// This allows for checking of ordered messages.
	metadata := map[string]string{
		messageKey: "test",
	}

// Send messages using the same metadata/message key so we can expect
// in-order processing.
Step("send and wait", sendRecvTest(metadata, consumerGroup1, consumerGroup2)).
```

sendRecvTest函数会往   pubsubName  "messagebus" 的 topic "neworder" 中发送 1000 条消息，而 app-1 启动后它的 sidecar dapr-1 时会订阅这个topic，然后发送给 app-1，app-1 通过 consumerGroup1 这个 watcher 记录接受到的消息，类似的 app-2 启动后它的 sidecar dapr-2 时也会订阅这个topic，然后发送给 app-2，app-2 通过 consumerGroup2 这个 watcher 记录接受到的消息，

由于 dapr-1 和 dapr-2 在订阅时使用了不同的 consumerGroup （kafkaCertification1 和 kafkaCertification2），因此每一条消息都会会分别发送到两个应用，一式两份。

### 运行简单无序测试

在启动第三个应用之后，运行简单无序测试：

```go
// 先重置 consumerGroup2
Step("reset", flow.Reset(consumerGroup2)).		
// Send messages with random keys to test message consumption
// across more than one consumer group and consumers per group.
Step("send and wait", sendRecvTest(map[string]string{}, consumerGroup2)).
```

由于 sidecar dapr-1 和 sidecar dapr-2 在订阅时使用的是同一个 consumerGroup （kafkaCertification2），因此消息会分流，两个应用各种接收到一部分。因为使用的都是同一个 watcher consumerGroup2，因此 consumerGroup2 里面应该接收到所有的消息，但是顺序无法保证。

### 运行重连测试

在收发消息的期间，关闭某一个broker，以测试组件重连的能力

```go
// Gradually stop each broker.
// This tests the components ability to handle reconnections
// when brokers are shutdown cleanly.
// 启动异步步骤，在后台发送消息
StepAsync("steady flow of messages to publish", &task,
          sendMessagesInBackground(consumerGroup1, consumerGroup2)).
// 5秒钟之后关闭 kafka1 broker
Step("wait", flow.Sleep(5*time.Second)).
Step("stop broker 1", dockercompose.Stop(clusterName, dockerComposeYAML, "kafka1")).

// 再过5秒钟之后关闭 kafka2 broker
Step("wait", flow.Sleep(5*time.Second)).
// Errors will likely start occurring here since quorum is lost.
Step("stop broker 2", dockercompose.Stop(clusterName, dockerComposeYAML, "kafka2")).

// 再过10秒钟之后关闭 kafka3 broker
Step("wait", flow.Sleep(10*time.Second)).
// Errors will definitely occur here.
Step("stop broker 3", dockercompose.Stop(clusterName, dockerComposeYAML, "kafka3")).

// 等待30秒后，重新启动 kafka1 / kafka2 / kafka3
Step("wait", flow.Sleep(30*time.Second)).
Step("restart broker 3", dockercompose.Start(clusterName, dockerComposeYAML, "kafka3")).
Step("restart broker 2", dockercompose.Start(clusterName, dockerComposeYAML, "kafka2")).
Step("restart broker 1", dockercompose.Start(clusterName, dockerComposeYAML, "kafka1")).

// Component should recover at this point.
// 再等30秒，组件应该能恢复工作
Step("wait", flow.Sleep(30*time.Second)).
// 检查消息已经都接收到
Step("assert messages", assertMessages(consumerGroup1, consumerGroup2)).
```



### 运行网络中断测试

在收发消息的期间，中断网络，以测试组件恢复的能力

```go
		// Simulate a network interruption.
		// This tests the components ability to handle reconnections
		// when Dapr is disconnected abnormally.
		StepAsync("steady flow of messages to publish", &task,
			sendMessagesInBackground(consumerGroup1, consumerGroup2)).
		Step("wait", flow.Sleep(5*time.Second)).
		//
		// Errors will occurring here.
		Step("interrupt network",
			network.InterruptNetwork(30*time.Second, nil, nil, "19092", "29092", "39092")).
		//
		// Component should recover at this point.
		Step("wait", flow.Sleep(30*time.Second)).
		Step("assert messages", assertMessages(consumerGroup1, consumerGroup2)).
```

备注: 网络中断的方式在我本地跑不起来



### 运行重平衡测试

验证多个consumer运行时关闭其中一个不会影响工作：

```go
// Reset and test that all messages are received during a
// consumer rebalance.
Step("reset", flow.Reset(consumerGroup2)).
// 后台异步发送消息
StepAsync("steady flow of messages to publish", &task,
          sendMessagesInBackground(consumerGroup2)).
// 15秒之后停止 sidecar dapr-2
Step("wait", flow.Sleep(15*time.Second)).
Step("stop sidecar 2", sidecar.Stop(sidecarName2)).
// 3秒之后停止 app-2
Step("wait", flow.Sleep(3*time.Second)).
Step("stop app 2", app.Stop(appID2)).
// 等待30秒
Step("wait", flow.Sleep(30*time.Second)).
// 预期消息还是能全部收到（app-3/dapr-3还在工作）
Step("assert messages", assertMessages(consumerGroup2)).
```

