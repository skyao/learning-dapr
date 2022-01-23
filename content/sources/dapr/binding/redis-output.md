---
type: docs
title: "资源绑定的Redis output实现源码分析"
linkTitle: "Redis output实现"
weight: 733
date: 2021-01-31
description: >
  Dapr资源绑定的Redis output实现源码分析
---


>  备注：根据 https://github.com/dapr/docs/blob/master/concepts/bindings/README.md 的描述，redis 只实现了 output binding。

### output binding 的实现

Redis的实现在 dapr/components-contrib 下，/bindings/redis/redis.go 中：

```go
func (r *Redis) Operations() []bindings.OperationKind {
  // 只支持create
	return []bindings.OperationKind{bindings.CreateOperation}
}

func (r *Redis) Invoke(req *bindings.InvokeRequest) (*bindings.InvokeResponse, error) {
  // 通过 metadata 传递 key
	if val, ok := req.Metadata["key"]; ok && val != "" {
		key := val
    // 调用标准 redis 客户端，执行 SET 命令
		_, err := r.client.DoContext(context.Background(), "SET", key, req.Data).Result()
		if err != nil {
			return nil, err
		}
		return nil, nil
	}
	return nil, errors.New("redis binding: missing key on write request metadata")
}
```



### 完整分析

初始化：

在 dapr runtime 初始化时，关联 redis 的 output binding实现：

```
bindings_loader.NewOutput("redis", func() bindings.OutputBinding {
   return redis.NewRedis(logContrib)
}),
```

然后 Init 方法会在 output binding初始化时被 dapr runtime 调用，Redis的实现内容为：

```go
// Init performs metadata parsing and connection creation
func (r *Redis) Init(meta bindings.Metadata) error {
  // 解析metadata
	m, err := r.parseMetadata(meta)
	if err != nil {
		return err
	}

  // redis 连接属性
	opts := &redis.Options{
		Addr:            m.host,
		Password:        m.password,
		DB:              defaultDB,
		MaxRetries:      m.maxRetries,
		MaxRetryBackoff: m.maxRetryBackoff,
	}

	/* #nosec */
	if m.enableTLS {
		opts.TLSConfig = &tls.Config{
			InsecureSkipVerify: m.enableTLS,
		}
	}

  // 建立redis连接
	r.client = redis.NewClient(opts)
	_, err = r.client.Ping().Result()
	if err != nil {
		return fmt.Errorf("redis binding: error connecting to redis at %s: %s", m.host, err)
	}

	return err
}
```

