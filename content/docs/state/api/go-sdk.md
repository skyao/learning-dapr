---
title: "状态管理API的go sdk封装"
linkTitle: "go sdk封装"
weight: 825
date: 2021-01-31
description: >
  Dapr状态管理API的go sdk封装
---

### go sdk使用案例

https://github.com/dapr/go-sdk 

对于简单场景，只要给出 store name / key / data 就好了：

```go
ctx := context.Background()
data := []byte("hello")
store := "my-store" // defined in the component YAML 

// save state with the key key1
if err := client.SaveState(ctx, store, "key1", data); err != nil {
    panic(err)
}

// get state for key key1
item, err := client.GetState(ctx, store, "key1")
if err != nil {
    panic(err)
}
fmt.Printf("data [key:%s etag:%s]: %s", item.Key, item.Etag, string(item.Value))

// delete state for key key1
if err := client.DeleteState(ctx, store, "key1"); err != nil {
    panic(err)
}
```

### get state

简单get方法，使用默认的并发选项：

```go
// GetState retreaves state from specific store using default consistency option.
func (c *GRPCClient) GetState(ctx context.Context, store, key string) (item *StateItem, err error) {
	return c.GetStateWithConsistency(ctx, store, key, StateConsistencyStrong)
}
```

但，默认并发选项是 StateConsistencyStrong，强一致性。

完整的get 方法：

```go
// GetStateWithConsistency retreaves state from specific store using provided state consistency.
func (c *GRPCClient) GetStateWithConsistency(ctx context.Context, store, key string, sc StateConsistency) (item *StateItem, err error) {
	if store == "" {
		return nil, errors.New("nil store")
	}
	if key == "" {
		return nil, errors.New("nil key")
	}

	req := &pb.GetStateRequest{
		StoreName:   store,
		Key:         key,
		Consistency: (v1.StateOptions_StateConsistency(sc)),
	}

	result, err := c.protoClient.GetState(authContext(ctx), req)
	if err != nil {
		return nil, errors.Wrap(err, "error getting state")
	}

	return &StateItem{
		Etag:  result.Etag,
		Key:   key,
		Value: result.Data,
	}, nil
}
```

基本上也没做什么。
