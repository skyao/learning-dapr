---
title: "dns"
linkTitle: "dns"
weight: 300
date: 2021-03-17
description: >
  dns 命名解析实现
---



### 实现

dns 的实现也是超级简单，类似 kubernetes 的实现，直接按照 DNS 的格式要求，评出一个 Kubernetes services 的 name 即可：

```go
// ResolveID resolves name to address in orchestrator.
func (k *resolver) ResolveID(req nameresolution.ResolveRequest) (string, error) {
	return fmt.Sprintf("%s-dapr.%s.svc:%d", req.ID, req.Namespace, req.Port), nil
}
```

所有参数都来自请求，只是拼接而已。



### 总结

DNS name resolver 返回的是一个简单的 Kubernetes services 的 name，形如 "app1-dapr.default.svc:80"。而不是一般意义上的 IP 地址。
