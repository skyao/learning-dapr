---
title: "资源绑定的初始化源码分析"
linkTitle: "初始化"
weight: 732
date: 2021-01-31
description: >
  Dapr资源绑定的初始化源码分析
---


## Binding Registry



### Binding Registry的初始化准备

Binding Registry 的初始化在 runtime 初始化时进行：

```go
func NewDaprRuntime(runtimeConfig *Config, globalConfig *config.Configuration) *DaprRuntime {
  ......
  bindingsRegistry:       bindings_loader.NewRegistry(),
}

func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {	
  ......
  a.bindingsRegistry.RegisterInputBindings(opts.inputBindings...)
	a.bindingsRegistry.RegisterOutputBindings(opts.outputBindings...)
  ......
}
```

这些 opts 来自 runtime 启动时的配置，如 cmd/daprd/main.go 下：

```go
func main() {
	rt, err := runtime.FromFlags()
	if err != nil {
		log.Fatal(err)
	}

	err = rt.Run(
    ......
    runtime.WithInputBindings(
			bindings_loader.NewInput("aws.sqs", func() bindings.InputBinding {
				return sqs.NewAWSSQS(logContrib)
			}),
			bindings_loader.NewInput("aws.kinesis", func() bindings.InputBinding {
				return kinesis.NewAWSKinesis(logContrib)
			}),
			bindings_loader.NewInput("azure.eventhubs", func() bindings.InputBinding {
				return eventhubs.NewAzureEventHubs(logContrib)
			}),
			bindings_loader.NewInput("kafka", func() bindings.InputBinding {
				return kafka.NewKafka(logContrib)
			}),
			bindings_loader.NewInput("mqtt", func() bindings.InputBinding {
				return mqtt.NewMQTT(logContrib)
			}),
			bindings_loader.NewInput("rabbitmq", func() bindings.InputBinding {
				return bindings_rabbitmq.NewRabbitMQ(logContrib)
			}),
			bindings_loader.NewInput("azure.servicebusqueues", func() bindings.InputBinding {
				return servicebusqueues.NewAzureServiceBusQueues(logContrib)
			}),
			bindings_loader.NewInput("azure.storagequeues", func() bindings.InputBinding {
				return storagequeues.NewAzureStorageQueues(logContrib)
			}),
			bindings_loader.NewInput("gcp.pubsub", func() bindings.InputBinding {
				return pubsub.NewGCPPubSub(logContrib)
			}),
			bindings_loader.NewInput("kubernetes", func() bindings.InputBinding {
				return kubernetes.NewKubernetes(logContrib)
			}),
			bindings_loader.NewInput("azure.eventgrid", func() bindings.InputBinding {
				return eventgrid.NewAzureEventGrid(logContrib)
			}),
			bindings_loader.NewInput("twitter", func() bindings.InputBinding {
				return twitter.NewTwitter(logContrib)
			}),
			bindings_loader.NewInput("cron", func() bindings.InputBinding {
				return cron.NewCron(logContrib)
			}),
		),
    runtime.WithOutputBindings(
			bindings_loader.NewOutput("aws.sqs", func() bindings.OutputBinding {
				return sqs.NewAWSSQS(logContrib)
			}),
			bindings_loader.NewOutput("aws.sns", func() bindings.OutputBinding {
				return sns.NewAWSSNS(logContrib)
			}),
			bindings_loader.NewOutput("aws.kinesis", func() bindings.OutputBinding {
				return kinesis.NewAWSKinesis(logContrib)
			}),
			bindings_loader.NewOutput("azure.eventhubs", func() bindings.OutputBinding {
				return eventhubs.NewAzureEventHubs(logContrib)
			}),
			bindings_loader.NewOutput("aws.dynamodb", func() bindings.OutputBinding {
				return dynamodb.NewDynamoDB(logContrib)
			}),
			bindings_loader.NewOutput("azure.cosmosdb", func() bindings.OutputBinding {
				return bindings_cosmosdb.NewCosmosDB(logContrib)
			}),
			bindings_loader.NewOutput("gcp.bucket", func() bindings.OutputBinding {
				return bucket.NewGCPStorage(logContrib)
			}),
			bindings_loader.NewOutput("http", func() bindings.OutputBinding {
				return http.NewHTTP(logContrib)
			}),
			bindings_loader.NewOutput("kafka", func() bindings.OutputBinding {
				return kafka.NewKafka(logContrib)
			}),
			bindings_loader.NewOutput("mqtt", func() bindings.OutputBinding {
				return mqtt.NewMQTT(logContrib)
			}),
			bindings_loader.NewOutput("rabbitmq", func() bindings.OutputBinding {
				return bindings_rabbitmq.NewRabbitMQ(logContrib)
			}),
			bindings_loader.NewOutput("redis", func() bindings.OutputBinding {
				return redis.NewRedis(logContrib)
			}),
			bindings_loader.NewOutput("aws.s3", func() bindings.OutputBinding {
				return s3.NewAWSS3(logContrib)
			}),
			bindings_loader.NewOutput("azure.blobstorage", func() bindings.OutputBinding {
				return blobstorage.NewAzureBlobStorage(logContrib)
			}),
			bindings_loader.NewOutput("azure.servicebusqueues", func() bindings.OutputBinding {
				return servicebusqueues.NewAzureServiceBusQueues(logContrib)
			}),
			bindings_loader.NewOutput("azure.storagequeues", func() bindings.OutputBinding {
				return storagequeues.NewAzureStorageQueues(logContrib)
			}),
			bindings_loader.NewOutput("gcp.pubsub", func() bindings.OutputBinding {
				return pubsub.NewGCPPubSub(logContrib)
			}),
			bindings_loader.NewOutput("azure.signalr", func() bindings.OutputBinding {
				return signalr.NewSignalR(logContrib)
			}),
			bindings_loader.NewOutput("twilio.sms", func() bindings.OutputBinding {
				return sms.NewSMS(logContrib)
			}),
			bindings_loader.NewOutput("twilio.sendgrid", func() bindings.OutputBinding {
				return sendgrid.NewSendGrid(logContrib)
			}),
			bindings_loader.NewOutput("azure.eventgrid", func() bindings.OutputBinding {
				return eventgrid.NewAzureEventGrid(logContrib)
			}),
			bindings_loader.NewOutput("cron", func() bindings.OutputBinding {
				return cron.NewCron(logContrib)
			}),
			bindings_loader.NewOutput("twitter", func() bindings.OutputBinding {
				return twitter.NewTwitter(logContrib)
			}),
			bindings_loader.NewOutput("influx", func() bindings.OutputBinding {
				return influx.NewInflux(logContrib)
			}),
		),
    ......
}
```

在这里配置各种 inputbinding 和 output binding的实现。

### Binding Registry的实现方式

pkg/components/bindings/registry.go，定义了多个数据结构：

```go
type (
	// InputBinding is an input binding component definition.
	InputBinding struct {
		Name          string
		FactoryMethod func() bindings.InputBinding
	}

	// OutputBinding is an output binding component definition.
	OutputBinding struct {
		Name          string
		FactoryMethod func() bindings.OutputBinding
	}

	// Registry is the interface of a components that allows callers to get registered instances of input and output bindings
	Registry interface {
		RegisterInputBindings(components ...InputBinding)
		RegisterOutputBindings(components ...OutputBinding)
		CreateInputBinding(name string) (bindings.InputBinding, error)
		CreateOutputBinding(name string) (bindings.OutputBinding, error)
	}

	bindingsRegistry struct {
		inputBindings  map[string]func() bindings.InputBinding
		outputBindings map[string]func() bindings.OutputBinding
	}
)
```

前面 runtime 初始化时，每个实现都通过 NewInput 方法和 NewOutput方法，将 name 和对应的InputBinding/OutputBinding关联起来：

```go
// NewInput creates a InputBinding.
func NewInput(name string, factoryMethod func() bindings.InputBinding) InputBinding {
	return InputBinding{
		Name:          name,
		FactoryMethod: factoryMethod,
	}
}

// NewOutput creates a OutputBinding.
func NewOutput(name string, factoryMethod func() bindings.OutputBinding) OutputBinding {
	return OutputBinding{
		Name:          name,
		FactoryMethod: factoryMethod,
	}
}
```

RegisterInputBindings 和 RegisterOutputBindings 方法用来注册 input binding 和 output binding

的实现，在runtime 初始化时被调用：

```go
// RegisterInputBindings registers one or more new input bindings.
func (b *bindingsRegistry) RegisterInputBindings(components ...InputBinding) {
	for _, component := range components {
		b.inputBindings[createFullName(component.Name)] = component.FactoryMethod
	}
}

// RegisterOutputBindings registers one or more new output bindings.
func (b *bindingsRegistry) RegisterOutputBindings(components ...OutputBinding) {
	for _, component := range components {
		b.outputBindings[createFullName(component.Name)] = component.FactoryMethod
	}
}

func createFullName(name string) string {
  // createFullName统一增加前缀 bindings.
	return fmt.Sprintf("bindings.%s", name)
}
```



## binding的初始化流程

pkg/runtime/runtime.go :

Binding 的初始化在 runtime 初始化时进行：

```
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
	......
	go a.processComponents()
	......
}
```

```
func (a *DaprRuntime) processComponents() {
   for {
      comp, more := <-a.pendingComponents
      if !more {
         a.pendingComponentsDone <- true
         return
      }
      if err := a.processOneComponent(comp); err != nil {
         log.Errorf("process component %s error, %s", comp.Name, err)
      }
   }
}
```

processOneComponent:

```go
func (a *DaprRuntime) processOneComponent(comp components_v1alpha1.Component) error {
	res := a.preprocessOneComponent(&comp)
  
	compCategory := a.figureOutComponentCategory(comp)

	......
	return nil
}
```

doProcessOneComponent:

```go
func (a *DaprRuntime) doProcessOneComponent(category ComponentCategory, comp components_v1alpha1.Component) error {
	switch category {
	case bindingsComponent:
		return a.initBinding(comp)
		......
	}
	return nil
}
```

initBinding:

```
func (a *DaprRuntime) initBinding(c components_v1alpha1.Component) error {
	if err := a.initOutputBinding(c); err != nil {
		log.Errorf("failed to init output bindings: %s", err)
		return err
	}

	if err := a.initInputBinding(c); err != nil {
		log.Errorf("failed to init input bindings: %s", err)
		return err
	}
	return nil
}
```

在这里进行 input binding 和 output binding 的初始化。

### Output Binding的初始化

pkg/runtime/runtime.go：

```go
func (a *DaprRuntime) initOutputBinding(c components_v1alpha1.Component) error {
  // 成功
	binding, err := a.bindingsRegistry.CreateOutputBinding(c.Spec.Type)
	if err != nil {
		log.Warnf("failed to create output binding %s (%s): %s", c.ObjectMeta.Name, c.Spec.Type, err)
		diag.DefaultMonitoring.ComponentInitFailed(c.Spec.Type, "creation")
		return err
	}

	if binding != nil {
		err := binding.Init(bindings.Metadata{
			Properties: a.convertMetadataItemsToProperties(c.Spec.Metadata),
			Name:       c.ObjectMeta.Name,
		})
		if err != nil {
			log.Errorf("failed to init output binding %s (%s): %s", c.ObjectMeta.Name, c.Spec.Type, err)
			diag.DefaultMonitoring.ComponentInitFailed(c.Spec.Type, "init")
			return err
		}
		log.Infof("successful init for output binding %s (%s)", c.ObjectMeta.Name, c.Spec.Type)
		a.outputBindings[c.ObjectMeta.Name] = binding
		diag.DefaultMonitoring.ComponentInitialized(c.Spec.Type)
	}
	return nil
}
```

其中 CreateOutputBinding 方法的实现在 `pkg/components/bindings/registry.go` 中：

```go
// Create instantiates an output binding based on `name`.
func (b *bindingsRegistry) CreateOutputBinding(name string) (bindings.OutputBinding, error) {
	if method, ok := b.outputBindings[name]; ok {
    // 调用 factory 方法生成具体实现的 outputBinding
		return method(), nil
	}
	return nil, errors.Errorf("couldn't find output binding %s", name)
}
```



### Input Binding的初始化

TODO