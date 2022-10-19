---
title: "命名解析概述"
linkTitle: "概述"
weight: 1
date: 2021-03-17
description: >
  命名解析
---

## 介绍

> Name resolvers provide a common way to interact with different name resolvers, which are used to return the address or IP of other services your applications may connect to.
>
> 命名解析器提供了一种与不同命名解析器互动的通用方法，这些解析器用于返回你的应用程序可能要连接到的其他服务的地址或IP。



### 接口定义

兼容的名称解析器需要实现 `nameresolution.go` 文件中的 `Resolver` 接口。

```go
// Resolver是命名解析器的接口。
type Resolver interface {
	// Init initializes name resolver.
	Init(metadata Metadata) error
	// ResolveID resolves name to address.
	ResolveID(req ResolveRequest) (string, error)
}
```



```go
// ResolveRequest 表示服务发现解析器请求。
type ResolveRequest struct {
	ID        string
	Namespace string
	Port      int
	Data      map[string]string
}
```

