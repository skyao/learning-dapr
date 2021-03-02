---
type: docs
title: "modes的源码学习"
linkTitle: "modes"
weight: 99
date: 2021-02-27
description: >
  Dapr modes package的源码学习
---

### 代码实现

modes 的代码超级简单，就一个 modes.go，内容也只有一点点：

```go
// DaprMode is the runtime mode for Dapr.
type DaprMode string

const (
	// KubernetesMode is a Kubernetes Dapr mode
	KubernetesMode DaprMode = "kubernetes"
	// StandaloneMode is a Standalone Dapr mode
	StandaloneMode DaprMode = "standalone"
)
```

Dapr有两种运行模式

- kubernetes 模式
- standalone 模式

### 运行模式的总结

两种模式的差异：

1. 配置文件读取的方式：

	- standalone 模式下读取本地文件，文件路径由命令行参数 `config` 指定。
	- kubernetes 模式下读取k8s中存储的CRD，CRD的名称由命令行参数 `config` 指定。

	```go
	config := flag.String("config", "", "Path to config file, or name of a configuration object")
	```

2. TODO

