---
type: docs
title: "patch_operation.go的源码学习"
linkTitle: "patch_operation.go"
weight: 20050
date: 2021-05-11
description: >
  Dapr Injector 中的 patch_operation.go 的 代码
---

代码非常简单，只定义了一个结构体 PatchOperation，用来表示要应用于Kubernetes资源的一个单独的变化。

```go
// PatchOperation represents a discreet change to be applied to a Kubernetes resource
type PatchOperation struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value interface{} `json:"value,omitempty"`
}
```

