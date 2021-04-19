---
type: docs
title: "main.go的源码学习"
linkTitle: "main.go"
weight: 20010
date: 2021-04-19
description: >
  Dapr Injector 的 main 代码
---

Dapr injector 中的 main.go 文件的源码分析。


## init() 方法

init() 进行初始化，包括 flag （logger， metric），

### flag 设定和读取

```go
func init() {
	loggerOptions := logger.DefaultOptions()
	// 这里设定了 `log-level` 和 `log-as-json`
	loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)

	metricsExporter := metrics.NewExporter(metrics.DefaultMetricNamespace)
	
	// 这里设定了 `metrics-port` 和 `enable-metrics`
metricsExporter.Options().AttachCmdFlags(flag.StringVar, flag.BoolVar)

	flag.Parse()
```

参考 injector pod yaml文件中 Command 段：

```yaml
    Command:
      /injector
    Args:
      --log-level
      info
      --log-as-json
      --enable-metrics
      --metrics-port
      9090
```

### 初始化 logger

```yaml
	// Apply options to all loggers
	if err := logger.ApplyOptionsToLoggers(&loggerOptions); err != nil {
		log.Fatal(err)
	} else {
		log.Infof("log level set to: %s", loggerOptions.OutputLevel)
	}

```

### 初始化 metrics

```go
	// Initialize dapr metrics exporter
	if err := metricsExporter.Init(); err != nil {
		log.Fatal(err)
	}

	// Initialize injector service metrics
	if err := monitoring.InitMetrics(); err != nil {
		log.Fatal(err)
	}
```

## main() 方法

### 获取配置

从环境变量中读取配置：

```go
func main() {
	log.Infof("starting Dapr Sidecar Injector -- version %s -- commit %s", version.Version(), version.Commit())

	ctx := signals.Context()
	cfg, err := injector.GetConfigFromEnvironment()
	if err != nil {
		log.Fatalf("error getting config: %s", err)
	}
	......
}
```

### 获取配置

```go
	kubeClient := utils.GetKubeClient()
	conf := utils.GetConfig()
	daprClient, _ := scheme.NewForConfig(conf)
```

### 启动 healthz 

```go
	go func() {
		healthzServer := health.NewServer(log)
		healthzServer.Ready()

		healthzErr := healthzServer.Run(ctx, healthzPort)
		if healthzErr != nil {
			log.Fatalf("failed to start healthz server: %s", healthzErr)
		}
	}()
```

### service account

```go
	uids, err := injector.AllowedControllersServiceAccountUID(ctx, kubeClient)
	if err != nil {
		log.Fatalf("failed to get authentication uids from services accounts: %s", err)
	}
```

### 创建 injector

```go
	injector.NewInjector(uids, cfg, daprClient, kubeClient).Run(ctx)
```

### graceful shutdown

```go
	shutdownDuration := 5 * time.Second
	log.Infof("allowing %s for graceful shutdown to complete", shutdownDuration)
	<-time.After(shutdownDuration)
```
