---
title: "consul"
linkTitle: "consul"
weight: 400
date: 2021-03-17
description: >
  consul 命名解析实现
---

### 初始化

初始化需要读取配置，建立连接：

```go
func (r *resolver) Init(metadata nr.Metadata) error {
	var err error

	r.config, err = getConfig(metadata)
	if err != nil {
		return err
	}

	if err = r.client.InitClient(r.config.Client); err != nil {
		return fmt.Errorf("failed to init consul client: %w", err)
	}

	// register service to consul
	......

	return nil
}
```

### 服务注册

在 init 函数中，还可以根据配置的要求执行 consul 的服务注册功能：

```go
	// register service to consul
	if r.config.Registration != nil {
		if err := r.client.Agent().ServiceRegister(r.config.Registration); err != nil {
			return fmt.Errorf("failed to register consul service: %w", err)
		}

		r.logger.Infof("service:%s registered on consul agent", r.config.Registration.Name)
	} else if _, err := r.client.Agent().Self(); err != nil {
		return fmt.Errorf("failed check on consul agent: %w", err)
	}
```

### 解析器实现

consul 命名解析器的实现比较简单：

```go
// ResolveID resolves name to address via consul.
func (r *resolver) ResolveID(req nr.ResolveRequest) (string, error) {
	cfg := r.config
  // 查询 consul 中对应服务的健康实例
  // 只用到 req.ID，namespace 没有用到
	services, _, err := r.client.Health().Service(req.ID, "", true, cfg.QueryOptions)
	if err != nil {
		return "", fmt.Errorf("failed to query healthy consul services: %w", err)
	}

	if len(services) == 0 {
		return "", fmt.Errorf("no healthy services found with AppID:%s", req.ID)
	}

  // shuffle：洗牌，将传入的 services 按照随机方式对调位置
	shuffle := func(services []*consul.ServiceEntry) []*consul.ServiceEntry {
		for i := len(services) - 1; i > 0; i-- {
			rndbig, _ := rand.Int(rand.Reader, big.NewInt(int64(i+1)))
			j := rndbig.Int64()

			services[i], services[j] = services[j], services[i]
		}

		return services
	}

  // 先洗牌，然后取结果中的第一个地址，相当于负载均衡中的随机算法
	svc := shuffle(services)[0]

	addr := ""

  // 取地址和port信息
	if port, ok := svc.Service.Meta[cfg.DaprPortMetaKey]; ok {
		if svc.Service.Address != "" {
			addr = fmt.Sprintf("%s:%s", svc.Service.Address, port)
		} else if svc.Node.Address != "" {
			addr = fmt.Sprintf("%s:%s", svc.Node.Address, port)
		} else {
			return "", fmt.Errorf("no healthy services found with AppID:%s", req.ID)
		}
	} else {
		return "", fmt.Errorf("target service AppID:%s found but DAPR_PORT missing from meta", req.ID)
	}

	return addr, nil
}
```

### 总结

consul name resolver 返回的是一个简单的ip/端口字符串，形如 "192.168.0.100:80"。对于多个实例，内部实现了随机算法。
