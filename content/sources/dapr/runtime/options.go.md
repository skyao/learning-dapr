---
type: docs
title: "options.go的源码学习"
linkTitle: "options.go"
weight: 1010
date: 2021-03-09
description: >
  用于定制 runtime 中包含的组件
---

Dapr runtime package中的 options.go 文件的源码学习，用于定制 runtime 中包含的组件。

### runtimeOpts 结构体定义

runtimeOpts封装了需要包含在 runtime 中的 component：

```go
type (
	// runtimeOpts encapsulates the components to include in the runtime.
	runtimeOpts struct {
		secretStores    []secretstores.SecretStore
		states          []state.State
		pubsubs         []pubsub.PubSub
		nameResolutions []nameresolution.NameResolution
		inputBindings   []bindings.InputBinding
		outputBindings  []bindings.OutputBinding
		httpMiddleware  []http.Middleware
	}
)
```

### Option 方法

Option 方法用于定制 runtime：

```go
type (
	// Option is a function that customizes the runtime.
	Option func(o *runtimeOpts)
)
```

### 定制runtime的With系列方法

提供多个 WithXxx() 方法，用于定制 runtime 的组件：

```go

// WithSecretStores adds secret store components to the runtime.
func WithSecretStores(secretStores ...secretstores.SecretStore) Option {
	return func(o *runtimeOpts) {
		o.secretStores = append(o.secretStores, secretStores...)
	}
}

// WithStates adds state store components to the runtime.
func WithStates(states ...state.State) Option {
	return func(o *runtimeOpts) {
		o.states = append(o.states, states...)
	}
}

// WithPubSubs adds pubsub store components to the runtime.
func WithPubSubs(pubsubs ...pubsub.PubSub) Option {
	return func(o *runtimeOpts) {
		o.pubsubs = append(o.pubsubs, pubsubs...)
	}
}

// WithNameResolutions adds name resolution components to the runtime.
func WithNameResolutions(nameResolutions ...nameresolution.NameResolution) Option {
	return func(o *runtimeOpts) {
		o.nameResolutions = append(o.nameResolutions, nameResolutions...)
	}
}

// WithInputBindings adds input binding components to the runtime.
func WithInputBindings(inputBindings ...bindings.InputBinding) Option {
	return func(o *runtimeOpts) {
		o.inputBindings = append(o.inputBindings, inputBindings...)
	}
}

// WithOutputBindings adds output binding components to the runtime.
func WithOutputBindings(outputBindings ...bindings.OutputBinding) Option {
	return func(o *runtimeOpts) {
		o.outputBindings = append(o.outputBindings, outputBindings...)
	}
}

// WithHTTPMiddleware adds HTTP middleware components to the runtime.
func WithHTTPMiddleware(httpMiddleware ...http.Middleware) Option {
	return func(o *runtimeOpts) {
		o.httpMiddleware = append(o.httpMiddleware, httpMiddleware...)
	}
}
```

这些方法都只是简单的往 runtimeOpts 结构体的各个组件字段里面保存信息，用于后续  runtime 的初始化。