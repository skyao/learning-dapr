---
type: docs
title: "options.go的源码学习"
linkTitle: "options.go"
weight: 9002
date: 2021-03-09
description: >
  metrics 相关的配置选项
---

Dapr metrics package中的 options.go文件的源码学习

## 代码实现

### Options 结构体定义

```go
// Options defines the sets of options for Dapr logging
type Options struct {
	// OutputLevel is the level of logging
	MetricsEnabled bool

	metricsPort string
}
```

### 默认值

metrics 默认端口 9090, 默认启用 metrics：

```go
const (
	defaultMetricsPort    = "9090"
	defaultMetricsEnabled = true
)

func defaultMetricOptions() *Options {
	return &Options{
		metricsPort:    defaultMetricsPort,
		MetricsEnabled: defaultMetricsEnabled,
	}
}
```

### MetricsPort() 方法实现

 MetricsPort() 方法用于获取 metrics 端口，如果配置错误，则使用默认端口 9090：

```go
// MetricsPort gets metrics port.
func (o *Options) MetricsPort() uint64 {
	port, err := strconv.ParseUint(o.metricsPort, 10, 64)
	if err != nil {
		// Use default metrics port as a fallback
		port, _ = strconv.ParseUint(defaultMetricsPort, 10, 64)
	}

	return port
}
```

## 解析命令行标记的方法

### AttachCmdFlags() 方法

AttachCmdFlags() 方法解析 metrics-port 和 enable-metrics 两个命令行标记：

```go
// AttachCmdFlags attaches metrics options to command flags
func (o *Options) AttachCmdFlags(
	stringVar func(p *string, name string, value string, usage string),
	boolVar func(p *bool, name string, value bool, usage string)) {
	stringVar(
		&o.metricsPort,
		"metrics-port",
		defaultMetricsPort,
		"The port for the metrics server")
	boolVar(
		&o.MetricsEnabled,
		"enable-metrics",
		defaultMetricsEnabled,
		"Enable prometheus metric")
}
```

### AttachCmdFlag() 方法

AttachCmdFlag() 方法只解析 metrics-port 命令行标记（不解析  enable-metrics ） ：

```go
// AttachCmdFlag attaches single metrics option to command flags
func (o *Options) AttachCmdFlag(
	stringVar func(p *string, name string, value string, usage string)) {
	stringVar(
		&o.metricsPort,
		"metrics-port",
		defaultMetricsPort,
		"The port for the metrics server")
}
```

## 使用场景

只解析 metrics-port 命令行标记 的 AttachCmdFlag() 方法在 dapr runtime 启动时被调用（也只被这一个地方调用）：

```go
metricsExporter := metrics.NewExporter(metrics.DefaultMetricNamespace)

// attaching only metrics-port option
metricsExporter.Options().AttachCmdFlag(flag.StringVar)
```

而解析 metrics-port 和 enable-metrics 两个命令行标记的 AttachCmdFlags() 方法被 injector / operator / placement / sentry 调用：

```go
func init() {
	metricsExporter := metrics.NewExporter(metrics.DefaultMetricNamespace)
	metricsExporter.Options().AttachCmdFlags(flag.StringVar, flag.BoolVar)
}
```