---
title: "registry.go的源码学习"
linkTitle: "registry.go"
weight: 10
date: 2023-08-24
description: >
  注册 workflow 组件
---



## 结构体定义

### Registry 结构体

Registry 结构体是用来注册返回工作流实现的组件接口

```go
import (
	wfs "github.com/dapr/components-contrib/workflows"
)
// Registry is an interface for a component that returns registered state store implementations.
type Registry struct {
	Logger             logger.Logger
	workflowComponents map[string]func(logger.Logger) wfs.Workflow
}
```

这里的 Workflow 在 components-contrib 中定义。

## 默认Registry

### 默认Registry的创建

package 中定义了一个 默认Registry， singleton, 还是 public的：

```go
// DefaultRegistry is the singleton with the registry .
var DefaultRegistry *Registry = NewRegistry()

// NewRegistry is used to create workflow registry.
func NewRegistry() *Registry {
	return &Registry{
		workflowComponents: map[string]func(logger.Logger) wfs.Workflow{},
	}
}
```

### RegisterComponent() 方法

RegisterComponent() 方法在在 register 结构体的 workflowComponents 字段中加入一条或多条记录

```go
func (s *Registry) RegisterComponent(componentFactory func(logger.Logger) wfs.Workflow, names ...string) {
	for _, name := range names {
		s.workflowComponents[createFullName(name)] = componentFactory
	}
}

func createFullName(name string) string {
	return strings.ToLower("workflow." + name)
}
```

key 是 `"workflow." + name` 转小写， value 是传入的 componentFactory，这是一个函数，只要传入一个 logger,就能返回 Workflow 实现。

### create() 方法

create() 方法根据指定的 name ，version 来构建对应的 workflow 实现：

```go
func (s *Registry) Create(name, version, logName string) (wfs.Workflow, error) {
	if method, ok := s.getWorkflowComponent(name, version, logName); ok {
		return method(), nil
	}
	return nil, fmt.Errorf("couldn't find wokflow %s/%s", name, version)
}
```

关键实现代码在 getWorkflowComponent() 方法中：

```go
func (s *Registry) getWorkflowComponent(name, version, logName string) (func() wfs.Workflow, bool) {
	nameLower := strings.ToLower(name)
	versionLower := strings.ToLower(version)
    // 用 nameLower+"/"+versionLower 拼接出 key
    // 然后在 register 结构体的 workflowComponents 字段中查找
    // TODO： 保存的时候是 key 是 `"workflow." + name` 转小写
	workflowFn, ok := s.workflowComponents[nameLower+"/"+versionLower]
	if ok {
		return s.wrapFn(workflowFn, logName), true
	}
    // 如果没有找到，看看是不是 InitialVersion
	if components.IsInitialVersion(versionLower) {
        // 如果是 InitialVersion，则不需要拼接 version 内容，直接通过 name 来查找
        // TODO：这要求 name 必须是 "workflow." 开头？
		workflowFn, ok = s.workflowComponents[nameLower]
		if ok {
			return s.wrapFn(workflowFn, logName), true
		}
	}
	return nil, false
}
```

如果有在 workflowComponents 字段中找到注册的 workflow 实现的 factory, 则用这个 factory 生成 workflow 的实现：

```go
func (s *Registry) wrapFn(componentFactory func(logger.Logger) wfs.Workflow, logName string) func() wfs.Workflow {
	return func() wfs.Workflow {
        // registey 的 logger 会被用来做 workflow 实现的 logger
		l := s.Logger
		if logName != "" && l != nil {
            // 在 logger 中增加 component 字段，值为 logName
			l = l.WithFields(map[string]any{
				"component": logName,
			})
		}
        // 最后调用 factory 的方法来构建 workflow 实现
		return componentFactory(l)
	}
}
```

## 总结

需要小心核对 key 的内容：

1. 是否带 "workflow." 前缀
2. 是否带version 或者 是否是 InitialVersion
