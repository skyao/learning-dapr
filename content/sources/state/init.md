---
type: docs
title: "状态管理的初始化源码分析"
linkTitle: "初始化"
weight: 832
date: 2021-01-31
description: >
  Dapr状态管理的初始化源码分析
---

## State Store Registry

### stateStoreRegistry的初始化准备

stateStoreRegistry Registry 的初始化在 runtime 初始化时进行：

```go
func NewDaprRuntime(runtimeConfig *Config, globalConfig *config.Configuration) *DaprRuntime {
  ......
  stateStoreRegistry:     state_loader.NewRegistry(),
}

func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {	
  ......
  a.stateStoreRegistry.Register(opts.states...)
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
    runtime.WithStates(
			state_loader.New("redis", func() state.Store {
				return state_redis.NewRedisStateStore(logContrib)
			}),
			state_loader.New("consul", func() state.Store {
				return consul.NewConsulStateStore(logContrib)
			}),
			state_loader.New("azure.blobstorage", func() state.Store {
				return state_azure_blobstorage.NewAzureBlobStorageStore(logContrib)
			}),
			state_loader.New("azure.cosmosdb", func() state.Store {
				return state_cosmosdb.NewCosmosDBStateStore(logContrib)
			}),
			state_loader.New("azure.tablestorage", func() state.Store {
				return state_azure_tablestorage.NewAzureTablesStateStore(logContrib)
			}),
			//state_loader.New("etcd", func() state.Store {
			//	return etcd.NewETCD(logContrib)
			//}),
			state_loader.New("cassandra", func() state.Store {
				return cassandra.NewCassandraStateStore(logContrib)
			}),
			state_loader.New("memcached", func() state.Store {
				return memcached.NewMemCacheStateStore(logContrib)
			}),
			state_loader.New("mongodb", func() state.Store {
				return mongodb.NewMongoDB(logContrib)
			}),
			state_loader.New("zookeeper", func() state.Store {
				return zookeeper.NewZookeeperStateStore(logContrib)
			}),
			state_loader.New("gcp.firestore", func() state.Store {
				return firestore.NewFirestoreStateStore(logContrib)
			}),
			state_loader.New("postgresql", func() state.Store {
				return postgresql.NewPostgreSQLStateStore(logContrib)
			}),
			state_loader.New("sqlserver", func() state.Store {
				return sqlserver.NewSQLServerStateStore(logContrib)
			}),
			state_loader.New("hazelcast", func() state.Store {
				return hazelcast.NewHazelcastStore(logContrib)
			}),
			state_loader.New("cloudstate.crdt", func() state.Store {
				return cloudstate.NewCRDT(logContrib)
			}),
			state_loader.New("couchbase", func() state.Store {
				return couchbase.NewCouchbaseStateStore(logContrib)
			}),
			state_loader.New("aerospike", func() state.Store {
				return aerospike.NewAerospikeStateStore(logContrib)
			}),
		),
    ......
}
```

在这里配置各种 state store 的实现。

### State Store Registry的实现方式

pkg/components/state/registry.go，定义了registry的接口和数据结构：

```go
// Registry is an interface for a component that returns registered state store implementations
type Registry interface {
	Register(components ...State)
	CreateStateStore(name string) (state.Store, error)
}

type stateStoreRegistry struct {
	stateStores map[string]func() state.Store
}
```

state.Store 是 dapr 定义的标准 state store的接口，所有的实现都要遵循这个接口。定义在 `github.com/dapr/components-contrib/state/store.go` 文件中：

```go
// Store is an interface to perform operations on store
type Store interface {
	Init(metadata Metadata) error
	Delete(req *DeleteRequest) error
	BulkDelete(req []DeleteRequest) error
	Get(req *GetRequest) (*GetResponse, error)
	Set(req *SetRequest) error
	BulkSet(req []SetRequest) error
}
```

前面 runtime 初始化时，每个实现都通过 New 方法将 name 和对应的 state store 关联起来：

```go
type State struct {
	Name          string
	FactoryMethod func() state.Store
}

func New(name string, factoryMethod func() state.Store) State {
	return State{
		Name:          name,
		FactoryMethod: factoryMethod,
	}
}
```

## State Store的初始化流程

pkg/runtime/runtime.go :

State 的初始化在 runtime 初始化时进行：

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
	case stateComponent:
		return a.initState(comp)
	}
		......
	return nil
}
```

initState方法的实现:

```
// Refer for state store api decision  https://github.com/dapr/dapr/blob/master/docs/decision_records/api/API-008-multi-state-store-api-design.md
func (a *DaprRuntime) initState(s components_v1alpha1.Component) error {
	// 构建 state store（这里才开始集成components的代码）
	store, err := a.stateStoreRegistry.CreateStateStore(s.Spec.Type)
	if err != nil {
		log.Warnf("error creating state store %s: %s", s.Spec.Type, err)
		diag.DefaultMonitoring.ComponentInitFailed(s.Spec.Type, "creation")
		return err
	}
	if store != nil {
		props := a.convertMetadataItemsToProperties(s.Spec.Metadata)
		// components的store实现在这里做初始化，如建连
		err := store.Init(state.Metadata{
			Properties: props,
		})
		if err != nil {
			diag.DefaultMonitoring.ComponentInitFailed(s.Spec.Type, "init")
			log.Warnf("error initializing state store %s: %s", s.Spec.Type, err)
			return err
		}

		// 将初始化完成的store实现存放在runtime中
		a.stateStores[s.ObjectMeta.Name] = store

		// set specified actor store if "actorStateStore" is true in the spec.
		actorStoreSpecified := props[actorStateStore]
		if actorStoreSpecified == "true" {
			if a.actorStateStoreCount++; a.actorStateStoreCount == 1 {
				a.actorStateStoreName = s.ObjectMeta.Name
			}
		}
		diag.DefaultMonitoring.ComponentInitialized(s.Spec.Type)
	}

	if a.actorStateStoreName == "" || a.actorStateStoreCount != 1 {
		log.Warnf("either no actor state store or multiple actor state stores are specified in the configuration, actor stores specified: %d", a.actorStateStoreCount)
	}

	return nil
}
```

其中 CreateStateStore 方法的实现在 `pkg/components/state/registry.go` 中：

```go
func (s *stateStoreRegistry) CreateStateStore(name string) (state.Store, error) {
	if method, ok := s.stateStores[name]; ok {
		return method(), nil
	}
	return nil, errors.Errorf("couldn't find state store %s", name)
}
```


