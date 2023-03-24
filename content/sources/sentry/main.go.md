---
title: "sentry的main函数入口"
linkTitle: "main函数入口"
weight: 10
date: 2022-02-22
description: >
  
---



sentry 模块的入口在文件 `cmd/sentry/main.go` 中。



## 准备工作

### 读取命令行参数



```go
const (
	defaultCredentialsPath = "/var/run/dapr/credentials"
	// defaultDaprSystemConfigName is the default resource object name for Dapr System Config.
	defaultDaprSystemConfigName = "daprsystem"

	healthzPort = 8080
)

func main() {
	configName := flag.String("config", defaultDaprSystemConfigName, "Path to config file, or name of a configuration object")
	credsPath := flag.String("issuer-credentials", defaultCredentialsPath, "Path to the credentials directory holding the issuer data")
	flag.StringVar(&credentials.RootCertFilename, "issuer-ca-filename", credentials.RootCertFilename, "Certificate Authority certificate filename")
	flag.StringVar(&credentials.IssuerCertFilename, "issuer-certificate-filename", credentials.IssuerCertFilename, "Issuer certificate filename")
	flag.StringVar(&credentials.IssuerKeyFilename, "issuer-key-filename", credentials.IssuerKeyFilename, "Issuer private key filename")
	trustDomain := flag.String("trust-domain", "localhost", "The CA trust domain")
	tokenAudience := flag.String("token-audience", "", "Expected audience for tokens; multiple values can be separated by a comma")
......
}
```

logger  和 metrics 的参数需要展开：

```go
	loggerOptions := logger.DefaultOptions()
	loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)

	metricsExporter := metrics.NewExporter(metrics.DefaultMetricNamespace)
	metricsExporter.Options().AttachCmdFlags(flag.StringVar, flag.BoolVar)
```

获取 k8s 的 配置文件路径：

```go
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
    // 读取 home 路径
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
    // 通过 `--kubeconfig` 传递完整的 kubeconfig 文件路径
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
```

最后解析一把：

```go
flag.Parse()
```

### 设置环境变量

将 kubeconfig 的值设置到 KUBE_CONFIG 环境变量：

```go
var (
	KubeConfigVar = "KUBE_CONFIG"
)

if err := utils.SetEnvVariables(map[string]string{
		utils.KubeConfigVar: *kubeconfig,
	}); err != nil {
		log.Fatalf("error set env failed:  %s", err.Error())
	}
```



## 初始化

这行日志标记着初始化正式开始：

```go
	log.Infof("starting sentry certificate authority -- version %s -- commit %s", buildinfo.Version(), buildinfo.Commit())
	log.Infof("log level set to: %s", loggerOptions.OutputLevel)
```

### 初始化metrics

```go
// Initialize dapr metrics exporter
	if err := metricsExporter.Init(); err != nil {
		log.Fatal(err)
	}
```

### 初始化监控

```go
	if err := monitoring.InitMetrics(); err != nil {
		log.Fatal(err)
	}
```



### 读取配置

```go
  // 拼凑文件路径
  issuerCertPath := filepath.Join(*credsPath, credentials.IssuerCertFilename) //issuer.crt
	issuerKeyPath := filepath.Join(*credsPath, credentials.IssuerKeyFilename)   // issuer.key
	rootCertPath := filepath.Join(*credsPath, credentials.RootCertFilename)     // ca.crt

  // 读取 sentry 配置：
  config, err := config.FromConfigName(*configName)
	if err != nil {
		log.Warn(err)
	}

  // 保存证书相关的各个路径和参数
	config.IssuerCertPath = issuerCertPath
	config.IssuerKeyPath = issuerKeyPath
	config.RootCertPath = rootCertPath
	config.TrustDomain = *trustDomain
	if *tokenAudience != "" {
		config.TokenAudience = tokenAudience
	}
```



## 启动服务



### 启动sentry server

```go
	ca := sentry.NewSentryCA()

	// Start the server in background
	err = ca.Start(runCtx, config)
	if err != nil {
		log.Fatalf("failed to restart sentry server: %s", err)
	}
```



### 启动 health server

```go
	log.Infof("starting watch on filesystem directory: %s", watchDir)

// Start the health server in background
	go func() {
		healthzServer := health.NewServer(log)
		healthzServer.Ready()

		if innerErr := healthzServer.Run(runCtx, healthzPort); innerErr != nil {
			log.Fatalf("failed to start healthz server: %s", innerErr)
		}
	}()
```



### 监控目录变化



```go
  issuerEvent := make(chan struct{})
  watchDir := filepath.Dir(config.IssuerCertPath)

  // Watch for changes in the watchDir
	// This also blocks until runCtx is canceled
	fswatcher.Watch(runCtx, watchDir, issuerEvent)
```

这个函数会一直阻塞直到 runCtx 被取消（这意味着要退出 sentry 进程）。

如果有文件更新，则 issuerEvent 会收到 event，issuerEvent 相关的处理代码：

```go
	go func() {
		// Restart the server when the issuer credentials change
		var restart <-chan time.Time
		for {
			select {
			case <-issuerEvent:
				monitoring.IssuerCertChanged()
				log.Debug("received issuer credentials changed signal")
				// Batch all signals within 2s of each other
				if restart == nil {
          // issuerEvent 不会被直接处理，而是安排在 2 秒发一个 restart event
          // 2秒之内的各种 issuerEvent 都会被这个 restart event 集中处理
					restart = time.After(2 * time.Second)
				}
			case <-restart:
        // 收到 restart，意味着 issuerEvent 已经积攒了 2 秒钟，可以统一处理了
				log.Warn("issuer credentials changed; reloading")
				innerErr := ca.Restart(runCtx, config)
				if innerErr != nil {
					log.Fatalf("failed to restart sentry server: %s", innerErr)
				}
        // 重置 restart，恢复原样，以便处理 2 秒之后的后续 issuerEvent
				restart = nil
			}
		}
	}()
```



## 退出



```go
	shutdownDuration := 5 * time.Second
	log.Infof("allowing %s for graceful shutdown to complete", shutdownDuration)
	<-time.After(shutdownDuration)
```



## 总结

去除非核心代码，sentry main 函数的主要功能是启动 sentry 的 ca server, 并监控目录，如果有变化则重启 ca server。
