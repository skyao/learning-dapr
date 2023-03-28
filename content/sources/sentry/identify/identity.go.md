---
title: "identify.go"
linkTitle: "identify.go"
weight: 10
date: 2022-02-22
description: >
  identify结构体定义
---



### 结构体定义

// Bundle 包含了足以以识别一个跨信任域和命名空间的工作负载的所有的元素：

```go
type Bundle struct {
	ID          string
	Namespace   string
	TrustDomain string
}
```

其实就三个元素： ID / Namespace 以及 TrustDomain

### NewBundle() 方法

NewBundle() 方法返回一个新的 identity Bundle。

```go
func NewBundle(id, namespace, trustDomain string) *Bundle {
	// Empty namespace and trust domain result in an empty bundle
  // 如果 namespace 或者 trust domain 为空，则返回空的 bundle（nil）
	if namespace == "" || trustDomain == "" {
		return nil
	}

  // 否则指示简单的赋值三个属性
	return &Bundle{
		ID:          id,
		Namespace:   namespace,
		TrustDomain: trustDomain,
	}
}
```

namespace和trustDomain是可选参数。当为空时，将返回一个 nil 值。
