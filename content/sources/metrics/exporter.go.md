---
type: docs
title: "exporter.go的源码学习"
linkTitle: "exporter.go"
weight: 9001
date: 2021-02-27
description: >
  Exporter 是用于 metrics 导出器的接口，当前只支持 Prometheus
---

Dapr metrics package中的 exporter.go文件的源码分析，包括结构体定义、方法实现。当前只支持 Prometheus。

## Exporter定义和实现

### Exporter 接口定义

Exporter 接口定义：

```go
// Exporter is the interface for metrics exporters
type Exporter interface {
	// Init initializes metrics exporter
	Init() error
	// Options returns Exporter options
	Options() *Options
}
```

### exporter 结构体定义

exporter 结构体定义：

```go
// exporter is the base struct
type exporter struct {
	namespace string
	options   *Options
	logger    logger.Logger
}
```

### 构建 exporter

```go
// NewExporter creates new MetricsExporter instance
func NewExporter(namespace string) Exporter {
	// TODO: support multiple exporters
	return &promMetricsExporter{
		&exporter{
			namespace: namespace,
			options:   defaultMetricOptions(),
			logger:    logger.NewLogger("dapr.metrics"),
		},
		nil,
	}
}
```

当前只支持 promMetrics 的 Exporter。

### 接口方法Options()的实现

Options() 方法简单返回 m.options：

```go
// Options returns current metric exporter options
func (m *exporter) Options() *Options {
	return m.options
}
```

具体的赋值在 defaultMetricOptions().

## Prometheus Exporter的实现

### promMetricsExporter 结构体定义

```go
// promMetricsExporter is prometheus metric exporter
type promMetricsExporter struct {
	*exporter
	ocExporter *ocprom.Exporter
}
```

内嵌 exporter （相当于继承），还有一个 ocprom.Exporter 字段。

### 接口方法 Init() 的实现

初始化 opencensus 的 exporter：

```go

// Init initializes opencensus exporter
func (m *promMetricsExporter) Init() error {
	if !m.exporter.Options().MetricsEnabled {
		return nil
	}

	// Add default health metrics for process
	
	// 添加默认的 health metrics： 进程信息，和 go 信息
	registry := prom.NewRegistry()
	registry.MustRegister(prom.NewProcessCollector(prom.ProcessCollectorOpts{}))
	registry.MustRegister(prom.NewGoCollector())

	var err error
	m.ocExporter, err = ocprom.NewExporter(ocprom.Options{
		Namespace: m.namespace,
		Registry:  registry,
	})

	if err != nil {
		return errors.Errorf("failed to create Prometheus exporter: %v", err)
	}

	// register exporter to view
	view.RegisterExporter(m.ocExporter)

	// start metrics server
	return m.startMetricServer()
}
```

### startMetricServer() 方法的实现

启动 MetricServer， 监听端口来自 options 的 MetricsPort，监听路径为 defaultMetricsPath:

```go

const (
	defaultMetricsPath     = "/"
)

// startMetricServer starts metrics server
func (m *promMetricsExporter) startMetricServer() error {
	if !m.exporter.Options().MetricsEnabled {
		// skip if metrics is not enabled
		return nil
	}

	addr := fmt.Sprintf(":%d", m.options.MetricsPort())

	if m.ocExporter == nil {
		return errors.New("exporter was not initialized")
	}

	m.exporter.logger.Infof("metrics server started on %s%s", addr, defaultMetricsPath)
	go func() {
		mux := http.NewServeMux()
		mux.Handle(defaultMetricsPath, m.ocExporter)

		if err := http.ListenAndServe(addr, mux); err != nil {
			m.exporter.logger.Fatalf("failed to start metrics server: %v", err)
		}
	}()

	return nil
}
```