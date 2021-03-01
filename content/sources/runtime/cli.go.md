---
type: docs
title: "cli.go的源码学习"
linkTitle: "cli.go"
weight: 1010
date: 2021-02-27
description: >
  解析命令行标记并返回 DaprRuntime 实例
---

Dapr runtime package中的 cli.go 文件的源码学习，解析命令行标记并返回 DaprRuntime 实例。

cli.go 基本上就一个 FromFlags() 方法。

## FromFlags()概述

FromFlags() 方法解析命令行标记并返回 DaprRuntime 实例：

```go
// FromFlags parses command flags and returns DaprRuntime instance
func FromFlags() (*DaprRuntime, error) {
   ......
   return NewDaprRuntime(runtimeConfig, globalConfig, accessControlList), nil
}
```

## 解析命令行标记

### 通用标记

代码如下：

```go
mode := flag.String("mode", string(modes.StandaloneMode), "Runtime mode for Dapr")
daprHTTPPort := flag.String("dapr-http-port", fmt.Sprintf("%v", DefaultDaprHTTPPort), "HTTP port for Dapr API to listen on")
daprAPIGRPCPort := flag.String("dapr-grpc-port", fmt.Sprintf("%v", DefaultDaprAPIGRPCPort), "gRPC port for the Dapr API to listen on")
daprInternalGRPCPort := flag.String("dapr-internal-grpc-port", "", "gRPC port for the Dapr Internal API to listen on")
appPort := flag.String("app-port", "", "The port the application is listening on")
profilePort := flag.String("profile-port", fmt.Sprintf("%v", DefaultProfilePort), "The port for the profile server")
appProtocol := flag.String("app-protocol", string(HTTPProtocol), "Protocol for the application: grpc or http")
componentsPath := flag.String("components-path", "", "Path for components directory. If empty, components will not be loaded. Self-hosted mode only")
config := flag.String("config", "", "Path to config file, or name of a configuration object")
appID := flag.String("app-id", "", "A unique ID for Dapr. Used for Service Discovery and state")
controlPlaneAddress := flag.String("control-plane-address", "", "Address for a Dapr control plane")
sentryAddress := flag.String("sentry-address", "", "Address for the Sentry CA service")
placementServiceHostAddr := flag.String("placement-host-address", "", "Addresses for Dapr Actor Placement servers")
allowedOrigins := flag.String("allowed-origins", cors.DefaultAllowedOrigins, "Allowed HTTP origins")
enableProfiling := flag.Bool("enable-profiling", false, "Enable profiling")
runtimeVersion := flag.Bool("version", false, "Prints the runtime version")
appMaxConcurrency := flag.Int("app-max-concurrency", -1, "Controls the concurrency level when forwarding requests to user code")
enableMTLS := flag.Bool("enable-mtls", false, "Enables automatic mTLS for daprd to daprd communication channels")
appSSL := flag.Bool("app-ssl", false, "Sets the URI scheme of the app to https and attempts an SSL connection")
daprHTTPMaxRequestSize := flag.Int("dapr-http-max-request-size", -1, "Increasing max size of request body in MB to handle uploading of big files. By default 4 MB.")
```

TODO：应该有命令行参数的文档，对照文档学习一遍。

### 解析日志相关的标记

```go
loggerOptions := logger.DefaultOptions()
loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)
```

### 解析metrics相关的标记

```go
metricsExporter := metrics.NewExporter(metrics.DefaultMetricNamespace)

// attaching only metrics-port option
metricsExporter.Options().AttachCmdFlag(flag.StringVar)
```

然后执行解析：

```go
flag.Parse()
```

### 执行version命令

如果只是version命令，则打印版本信息之后就可以退出进程了：

```go
runtimeVersion := flag.Bool("version", false, "Prints the runtime version")

if *runtimeVersion {
   fmt.Println(version.Version())
   os.Exit(0)
}
```



## 初始化日志和metrics

### 日志初始化

根据日志属性初始化logger:

```go
loggerOptions := logger.DefaultOptions()
loggerOptions.AttachCmdFlags(flag.StringVar, flag.BoolVar)

if *appID == "" {
   return nil, errors.New("app-id parameter cannot be empty")
}

// Apply options to all loggers
loggerOptions.SetAppID(*appID)
if err := logger.ApplyOptionsToLoggers(&loggerOptions); err != nil {
   return nil, err
}
```

完成日志初始化之后就可以愉快的打印日志了：

```go
log.Infof("starting Dapr Runtime -- version %s -- commit %s", version.Version(), version.Commit())
log.Infof("log level set to: %s", loggerOptions.OutputLevel)
```

### metrics初始化

初始化dapr metrics exporter：

```go
// Initialize dapr metrics exporter
if err := metricsExporter.Init(); err != nil {
   log.Fatal(err)
}
```

## 解析配置

### 解析dapr各种端口设置

dapr-http-port / dapr-grpc-port / profile-port / dapr-internal-grpc-port / app-port ：

```go
daprHTTP, err := strconv.Atoi(*daprHTTPPort)
if err != nil {
   return nil, errors.Wrap(err, "error parsing dapr-http-port flag")
}

daprAPIGRPC, err := strconv.Atoi(*daprAPIGRPCPort)
if err != nil {
   return nil, errors.Wrap(err, "error parsing dapr-grpc-port flag")
}

profPort, err := strconv.Atoi(*profilePort)
if err != nil {
   return nil, errors.Wrap(err, "error parsing profile-port flag")
}

var daprInternalGRPC int
if *daprInternalGRPCPort != "" {
   daprInternalGRPC, err = strconv.Atoi(*daprInternalGRPCPort)
   if err != nil {
      return nil, errors.Wrap(err, "error parsing dapr-internal-grpc-port")
   }
} else {
   daprInternalGRPC, err = grpc.GetFreePort()
   if err != nil {
      return nil, errors.Wrap(err, "failed to get free port for internal grpc server")
   }
}

var applicationPort int
if *appPort != "" {
   applicationPort, err = strconv.Atoi(*appPort)
   if err != nil {
      return nil, errors.Wrap(err, "error parsing app-port")
   }
}
```

### 解析其他配置

继续解析 maxRequestBodySize / placementAddresses / concurrency / appProtocol 等 配置：

```go
var maxRequestBodySize int
if *daprHTTPMaxRequestSize != -1 {
   maxRequestBodySize = *daprHTTPMaxRequestSize
} else {
   maxRequestBodySize = DefaultMaxRequestBodySize
}

placementAddresses := []string{}
if *placementServiceHostAddr != "" {
   placementAddresses = parsePlacementAddr(*placementServiceHostAddr)
}

var concurrency int
if *appMaxConcurrency != -1 {
   concurrency = *appMaxConcurrency
}

appPrtcl := string(HTTPProtocol)
if *appProtocol != string(HTTPProtocol) {
   appPrtcl = *appProtocol
}
```

### 构建Runtime的三大配置

### 构建runtimeConfig

```go
runtimeConfig := NewRuntimeConfig(*appID, placementAddresses, *controlPlaneAddress, *allowedOrigins, *config, *componentsPath,
   appPrtcl, *mode, daprHTTP, daprInternalGRPC, daprAPIGRPC, applicationPort, profPort, *enableProfiling, concurrency, *enableMTLS, *sentryAddress, *appSSL, maxRequestBodySize)
```

MTLS相关的配置：

```go
if *enableMTLS {
   runtimeConfig.CertChain, err = security.GetCertChain()
   if err != nil {
      return nil, err
   }
}
```

### 构建globalConfig

```go
var globalConfig *global_config.Configuration
```

根据 config 配置文件的配置，还有 dapr 模式的配置，读取相应的配置文件：

```go
config := flag.String("config", "", "Path to config file, or name of a configuration object")

if *config != "" {
   switch modes.DaprMode(*mode) {
      case modes.KubernetesMode:
      client, conn, clientErr := client.GetOperatorClient(*controlPlaneAddress, security.TLSServerName, runtimeConfig.CertChain)
      if clientErr != nil {
         return nil, clientErr
      }
      defer conn.Close()
      namespace = os.Getenv("NAMESPACE")
      globalConfig, configErr = global_config.LoadKubernetesConfiguration(*config, namespace, client)
      case modes.StandaloneMode:
      globalConfig, _, configErr = global_config.LoadStandaloneConfiguration(*config)
   }

   if configErr != nil {
      log.Debugf("Config error: %v", configErr)
   }
}

if configErr != nil {
   log.Fatalf("error loading configuration: %s", configErr)
}
```

简单说：kubernetes 模式下读取CRD，standalone 模式下读取本地配置文件。

如果 config 没有配置，则使用默认的 global 配置：

```go
if globalConfig == nil {
   log.Info("loading default configuration")
   globalConfig = global_config.LoadDefaultConfiguration()
}
```

### 构建accessControlList

```go
var accessControlList *global_config.AccessControlList

accessControlList, err = global_config.ParseAccessControlSpec(globalConfig.Spec.AccessControlSpec, string(runtimeConfig.ApplicationProtocol))
if err != nil {
   log.Fatalf(err.Error())
}
```

## 构造 DaprRuntime 实例

最后构造 DaprRuntime 实例：

```go
return NewDaprRuntime(runtimeConfig, globalConfig, accessControlList), nil
```