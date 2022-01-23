---
type: docs
title: "config.go的源码学习"
linkTitle: "config.go"
weight: 1012
date: 2021-03-12
description: >
  解析命令行标记并返回 DaprRuntime 实例
---

Dapr runtime package中的 cli.go 文件的源码学习，解析命令行标记并返回 DaprRuntime 实例。

cli.go 基本上就一个 FromFlags() 方法。

### 常量定义

protocol，目前只支持 http 和 grpc ：

```go
// Protocol is a communications protocol
type Protocol string

const (
	// GRPCProtocol is a gRPC communication protocol
	GRPCProtocol Protocol = "grpc"
	// HTTPProtocol is a HTTP communication protocol
	HTTPProtocol Protocol = "http"
)
```

各种端口的默认值：

```go
const (
	// DefaultDaprHTTPPort is the default http port for Dapr
	DefaultDaprHTTPPort = 3500
	// DefaultDaprAPIGRPCPort is the default API gRPC port for Dapr
	DefaultDaprAPIGRPCPort = 50001
	// DefaultProfilePort is the default port for profiling endpoints
	DefaultProfilePort = 7777
	// DefaultMetricsPort is the default port for metrics endpoints
	DefaultMetricsPort = 9090
)
```

http默认配置，目前只有一个 MaxRequestBodySize ：

```go
const (
	// DefaultMaxRequestBodySize is the default option for the maximum body size in MB for Dapr HTTP servers
	DefaultMaxRequestBodySize = 4
)
```

### Config 结构体

```go
// Config holds the Dapr Runtime configuration
type Config struct {
	ID                   string
	HTTPPort             int
	ProfilePort          int
	EnableProfiling      bool
	APIGRPCPort          int
	InternalGRPCPort     int
	ApplicationPort      int
	ApplicationProtocol  Protocol
	Mode                 modes.DaprMode
	PlacementAddresses   []string
	GlobalConfig         string
	AllowedOrigins       string
	Standalone           config.StandaloneConfig
	Kubernetes           config.KubernetesConfig
	MaxConcurrency       int
	mtlsEnabled          bool
	SentryServiceAddress string
	CertChain            *credentials.CertChain
	AppSSL               bool
	MaxRequestBodySize   int
}
```

> 有点乱，所有的字段都是扁平的，以后越加越多。。。

### 构建Config

简单赋值构建 config 结构体，这个参数是在太多了一点：

```go
// NewRuntimeConfig returns a new runtime config
func NewRuntimeConfig(
   id string, placementAddresses []string,
   controlPlaneAddress, allowedOrigins, globalConfig, componentsPath, appProtocol, mode string,
   httpPort, internalGRPCPort, apiGRPCPort, appPort, profilePort int,
   enableProfiling bool, maxConcurrency int, mtlsEnabled bool, sentryAddress string, appSSL bool, maxRequestBodySize int) *Config {
   return &Config{
      ID:                  id,
      HTTPPort:            httpPort,
      InternalGRPCPort:    internalGRPCPort,
      APIGRPCPort:         apiGRPCPort,
      ApplicationPort:     appPort,
      ProfilePort:         profilePort,
      ApplicationProtocol: Protocol(appProtocol),
      Mode:                modes.DaprMode(mode),
      PlacementAddresses:  placementAddresses,
      GlobalConfig:        globalConfig,
      AllowedOrigins:      allowedOrigins,
      Standalone: config.StandaloneConfig{
         ComponentsPath: componentsPath,
      },
      Kubernetes: config.KubernetesConfig{
         ControlPlaneAddress: controlPlaneAddress,
      },
      EnableProfiling:      enableProfiling,
      MaxConcurrency:       maxConcurrency,
      mtlsEnabled:          mtlsEnabled,
      SentryServiceAddress: sentryAddress,
      AppSSL:               appSSL,
      MaxRequestBodySize:   maxRequestBodySize,
   }
}
```