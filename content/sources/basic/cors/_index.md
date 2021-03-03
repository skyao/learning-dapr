---
type: docs
title: "cors的源码学习"
linkTitle: "cors"
weight: 330
date: 2021-02-27
description: >
  Dapr cors package的源码学习
---

### 代码实现

cors 的代码超级简单，就一个 cors.go，内容也只有一点点：

```go
// DefaultAllowedOrigins is the default origins allowed for the Dapr HTTP servers
const DefaultAllowedOrigins = "*"
```

### AllowedOrigins配置的读取

AllowedOrigins 配置在启动时通过命令行参数 `allowed-origins` 传入，默认值为 DefaultAllowedOrigins （"*"）。然后传入给 NewRuntimeConfig（）方法：

```go
func FromFlags() (*DaprRuntime, error) {
allowedOrigins := flag.String("allowed-origins", cors.DefaultAllowedOrigins, "Allowed HTTP origins")

	runtimeConfig := NewRuntimeConfig(*appID, placementAddresses, *controlPlaneAddress, *allowedOrigins ......)
}

```

之后保存在 NewRuntimeConfig 的 AllowedOrigins 字段中：

```go
func NewRuntimeConfig(
   id string, placementAddresses []string,
   controlPlaneAddress, allowedOrigins ......) *Config {
   return &Config{
   	AllowedOrigins:      allowedOrigins,
   	......
   }
```



### AllowedOrigins配置的使用

`pkg/http/server.go` 的 useCors() 方法：

```go
func (s *server) useCors(next fasthttp.RequestHandler) fasthttp.RequestHandler {
   if s.config.AllowedOrigins == cors_dapr.DefaultAllowedOrigins {
      return next
   }

   log.Infof("enabled cors http middleware")
   origins := strings.Split(s.config.AllowedOrigins, ",")
   corsHandler := s.getCorsHandler(origins)
   return corsHandler.CorsMiddleware(next)
}
```