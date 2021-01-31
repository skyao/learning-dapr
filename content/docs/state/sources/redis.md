---
title: "状态管理中Redis实现的处理源码分析"
linkTitle: "Redis实现"
weight: 834
date: 2021-01-31
description: >
  Dapr状态管理中Redis实现的处理源码分析
---

### 状态管理的redis实现

Redis的实现在 dapr/components-contrib 下，/state/redis/redis.go 中：

```go
// StateStore is a Redis state store
type StateStore struct {
	client   *redis.Client
	json     jsoniter.API
	metadata metadata
	replicas int

	logger logger.Logger
}

// NewRedisStateStore returns a new redis state store
func NewRedisStateStore(logger logger.Logger) *StateStore {
	return &StateStore{
		json:   jsoniter.ConfigFastest,
		logger: logger,
	}
}
```

### 初始化

在 dapr runtime 初始化时，关联 redis 的 state 实现：

```
state_loader.New("redis", func() state.Store {
    return state_redis.NewRedisStateStore(logContrib)
}),
```

然后 Init 方法会在 state 初始化时被 dapr runtime 调用，Redis的实现内容为：

```go
// Init does metadata and connection parsing
func (r *StateStore) Init(metadata state.Metadata) error {
	m, err := parseRedisMetadata(metadata)
	if err != nil {
		return err
	}
	r.metadata = m

	if r.metadata.failover {
		r.client = r.newFailoverClient(m)
	} else {
		r.client = r.newClient(m)
	}

	if _, err = r.client.Ping().Result(); err != nil {
		return fmt.Errorf("redis store: error connecting to redis at %s: %s", m.host, err)
	}

	r.replicas, err = r.getConnectedSlaves()

	return err
}
```



### get state

get的实现方式：

```go
// Get retrieves state from redis with a key
func (r *StateStore) Get(req *state.GetRequest) (*state.GetResponse, error) {
   res, err := r.client.DoContext(context.Background(), "HGETALL", req.Key).Result() // Prefer values with ETags
   if err != nil {
      return r.directGet(req) //Falls back to original get
   }
   if res == nil {
      // 结果为空的处理1
      return &state.GetResponse{}, nil
   }
   vals := res.([]interface{})
   if len(vals) == 0 {
      // 结果为空的处理2
      // 所以如果没有找到对应key的值，是给空应答，而不是报错
      return &state.GetResponse{}, nil
   }

   data, version, err := r.getKeyVersion(vals)
   if err != nil {
      return nil, err
   }
   return &state.GetResponse{
      Data: []byte(data),
      ETag: version,
   }, nil
}
```

#### 支持ETag的实现方式

要支持ETag，就不能简单用 redis 的 key / value 方式直接在value中存放state的数据（data字段，byte[]格式），这个“value”需要包含出data之外的其他Etag字段，比如 version。

redis state实现的设计方式方式是：对于每个存储在 redis 中的 state item中，其value是一个hashmap，在这个value hashmap中通过不同的key存放多个信息：

- data：state的数据
- version：ETag需要的version

所以前面要用 **HGETALL** 命令把这个hashamap的所有key/value都取出来，然后现在要通过getKeyVersion方法来从这些key/value中读取data和version：

```go
func (r *StateStore) getKeyVersion(vals []interface{}) (data string, version string, err error) {
   seenData := false
   seenVersion := false
   for i := 0; i < len(vals); i += 2 {
      field, _ := strconv.Unquote(fmt.Sprintf("%q", vals[i]))
      switch field {
      case "data":
         data, _ = strconv.Unquote(fmt.Sprintf("%q", vals[i+1]))
         seenData = true
      case "version":
         version, _ = strconv.Unquote(fmt.Sprintf("%q", vals[i+1]))
         seenVersion = true
      }
   }
   if !seenData || !seenVersion {
      return "", "", errors.New("required hash field 'data' or 'version' was not found")
   }
   return data, version, nil
}
```

返回的时候，带上ETag：

```go
return &state.GetResponse{
      Data: []byte(data),
      ETag: version,
   }, nil
```

#### 不支持ETag的实现方式

如果 HGETALL 命令执行失败，则fall back到普通场景：redis中只简单保存数据，没有etag。此时保存方式就是简单的key/value，用简单的 GET 命令直接读取：

```go
func (r *StateStore) directGet(req *state.GetRequest) (*state.GetResponse, error) {
   res, err := r.client.DoContext(context.Background(), "GET", req.Key).Result()
   if err != nil {
      return nil, err
   }

   if res == nil {
      return &state.GetResponse{}, nil
   }

   s, _ := strconv.Unquote(fmt.Sprintf("%q", res))
   return &state.GetResponse{
      Data: []byte(s),
   }, nil
}
```



备注：这个设计有个性能问题，如果redis中的数据是用简单key/value存储，没有etag，则每次读取都要进行两个：第一次 HGETALL 命令失败，然后 fall back 用 GET 命令再读第二次。

### save state

redis的实现，有 set 方法和 BulkSet

```go
// Set saves state into redis
func (r *StateStore) Set(req *state.SetRequest) error {
   return state.SetWithOptions(r.setValue, req)
}

// BulkSet performs a bulks save operation
func (r *StateStore) BulkSet(req []state.SetRequest) error {
   for i := range req {
      err := r.Set(&req[i])
      if err != nil {
         // 这个地方有异议
         // 按照代码逻辑，只要有一个save操作失败，就直接return而放弃后续的操作
         return err
      }
   }

   return nil
}
```

实际实现在 r.setValue 方法中：

```
func (r *StateStore) setValue(req *state.SetRequest) error {
   err := state.CheckRequestOptions(req.Options)
   if err != nil {
      return err
   }
   
   // 解析etag，要求etag必须是可以转为整型
   ver, err := r.parseETag(req.ETag)
   if err != nil {
      return err
   }

   // LastWrite win意味着无视ETag的异同，强制写入
   // 所以这里重置 ver 为 0
   if req.Options.Concurrency == state.LastWrite {
      ver = 0
   }

   bt, _ := utils.Marshal(req.Value, r.json.Marshal)

	 // 用 EVAL 命令执行一段 LUA 脚本，脚本内容为 setQuery
   _, err = r.client.DoContext(context.Background(), "EVAL", setQuery, 1, req.Key, ver, bt).Result()
   if err != nil {
      return fmt.Errorf("failed to set key %s: %s", req.Key, err)
   }

	 // 如果要求强一致性，而且副本数量大于0
   if req.Options.Consistency == state.Strong && r.replicas > 0 {
     // 则需要等待所有副本数都写入成功
      _, err = r.client.DoContext(context.Background(), "WAIT", r.replicas, 1000).Result()
      if err != nil {
         return fmt.Errorf("timed out while waiting for %v replicas to acknowledge write", r.replicas)
      }
   }

   return nil
}
```

更多redis细节：

- setQuery 脚本

```
setQuery                 = "local var1 = redis.pcall(\"HGET\", KEYS[1], \"version\"); if type(var1) == \"table\" then redis.call(\"DEL\", KEYS[1]); end; if not var1 or type(var1)==\"table\" or var1 == \"\" or var1 == ARGV[1] or ARGV[1] == \"0\" then redis.call(\"HSET\", KEYS[1], \"data\", ARGV[2]) return redis.call(\"HINCRBY\", KEYS[1], \"version\", 1) else return error(\"failed to set key \" .. KEYS[1]) end"
```

- WAIT numreplicas timeout 命令：https://redis.io/commands/wait



### delete state

```go
// Delete performs a delete operation
func (r *StateStore) Delete(req *state.DeleteRequest) error {
   err := state.CheckRequestOptions(req.Options)
   if err != nil {
      return err
   }
   return state.DeleteWithOptions(r.deleteValue, req)
}

// 内部循环调用 Delete
// BulkDelete 方法没有暴露给 dapr runtime
// BulkDelete performs a bulk delete operation
func (r *StateStore) BulkDelete(req []state.DeleteRequest) error {
   for i := range req {
      err := r.Delete(&req[i])
      if err != nil {
         return err
      }
   }

   return nil
}
```

实际实现在 r.deleteValue 方法中：

```
func (r *StateStore) deleteValue(req *state.DeleteRequest) error {
   if req.ETag == "" {
      // ETag的空值则改为 “0” / 零值
      req.ETag = "0"
   }
   _, err := r.client.DoContext(context.Background(), "EVAL", delQuery, 1, req.Key, req.ETag).Result()

   if err != nil {
      return fmt.Errorf("failed to delete key '%s' due to ETag mismatch", req.Key)
   }

   return nil
}
```

更多redis细节：

- delQuery 脚本

```
delQuery                 = "local var1 = redis.pcall(\"HGET\", KEYS[1], \"version\"); if not var1 or type(var1)==\"table\" or var1 == ARGV[1] or var1 == \"\" or ARGV[1] == \"0\" then return redis.call(\"DEL\", KEYS[1]) else return error(\"failed to delete \" .. KEYS[1]) end"
```

###  State Transaction

redis state store 实现了 TransactionalStore，它的 Multi方式：

```go
// Multi performs a transactional operation. succeeds only if all operations succeed, and fails if one or more operations fail
func (r *StateStore) Multi(request *state.TransactionalStateRequest) error {
   // 用的是 redis-go 封装的 TxPipeline
   pipe := r.client.TxPipeline()
   for _, o := range request.Operations {
      if o.Operation == state.Upsert {
         req := o.Request.(state.SetRequest)

         bt, _ := utils.Marshal(req.Value, r.json.Marshal)

         pipe.Set(req.Key, bt, defaultExpirationTime)
      } else if o.Operation == state.Delete {
         req := o.Request.(state.DeleteRequest)
         pipe.Del(req.Key)
      }
   }

   _, err := pipe.Exec()
   return err
}
```