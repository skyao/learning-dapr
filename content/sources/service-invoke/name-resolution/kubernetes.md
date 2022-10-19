---
title: "kubernetes"
linkTitle: "kubernetes"
weight: 200
date: 2021-03-17
description: >
  kubernetes 命名解析实现
---



### 实现

kubernetes 的实现超级简单，直接按照 Kubernetes services 的格式要求，评出一个 Kubernetes services 的 name 即可：

```go
// ResolveID resolves name to address in Kubernetes.
func (k *resolver) ResolveID(req nameresolution.ResolveRequest) (string, error) {
	// Dapr requires this formatting for Kubernetes services
	return fmt.Sprintf("%s-dapr.%s.svc.%s:%d", req.ID, req.Namespace, k.clusterDomain, req.Port), nil
}
```

其中， req.ID 和 req.Namespace 对应到 Kubernetes 的 service name 和 namespace，注意这里的 Kubernetes service 是在 ID 后面加了 "-dapr" 后缀。Port 来自请求参数，简单拼接而已。

### clusterDomain 的设置

clusterDomain 稍微复杂一点，默认值是 "cluster.local"，在构建 Resolver 时设置：

```go
const (
	DefaultClusterDomain = "cluster.local"
)

type resolver struct {
	logger        logger.Logger
	clusterDomain string
}

// NewResolver creates Kubernetes name resolver.
func NewResolver(logger logger.Logger) nameresolution.Resolver {
	return &resolver{
		logger:        logger,
		clusterDomain: DefaultClusterDomain,
	}
}
```

可以在配置中设置名为 "clusterDomain" 的 metadata 来覆盖默认值：

```go
const (
	ClusterDomainKey     = "clusterDomain"
)

func (k *resolver) Init(metadata nameresolution.Metadata) error {
	configInterface, err := config.Normalize(metadata.Configuration)
	if err != nil {
		return err
	}
	if config, ok := configInterface.(map[string]string); ok {
		clusterDomain := config[ClusterDomainKey]
		if clusterDomain != "" {
			k.clusterDomain = clusterDomain
		}
	}

	return nil
}
```



### 总结

kubernetes name resolver 返回的是一个简单的 Kubernetes services 的 name，形如 "app1-dapr.default.svc.cluster.local:80"。而不是一般意义上的 IP 地址。
