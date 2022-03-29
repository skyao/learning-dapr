---
type: docs
title: "HTTP 服务器概述"
linkTitle: "概述"
weight: 1
date: 2021-01-29
description: >
  dapr runtime 中的 HTTP 服务器
---

## 介绍

Dapr runtime 对外提供 Dapr HTTP API，对外暴露的端口，默认是 **3500**。


```plantuml
title Service Invoke via HTTP
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    client
]
participant SDK_client [
    =SDK
    ----
    client
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant SDK_server [
    =SDK
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]
end box

user_code_client -> SDK_client : Invoke\nService() 
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```

## 实现

dapr runtime 采用 fasthttp 做 HTTP 服务器,引入的依赖如下 go.mod 文件中所示:

```bash
	github.com/valyala/fasthttp v1.31.1-0.20211216042702-258a4c17b4f4
	github.com/AdhityaRamadhanus/fasthttpcors v0.0.0-20170121111917-d4c07198763a
	github.com/fasthttp/router v1.3.8
	github.com/fasthttp-contrib/sessions v0.0.0-20160905201309-74f6ac73d5d5 // indirect
```

### 启动 HTTP server

在 dapr runtime 启动时的初始化过程中，会启动 HTTP server， 代码在 `pkg/runtime/runtime.go` 中：

```go
func (a *DaprRuntime) initRuntime(opts *runtimeOpts) error {
  ......
  // Start HTTP Server
	err = a.startHTTPServer(a.runtimeConfig.HTTPPort, a.runtimeConfig.PublicPort, a.runtimeConfig.ProfilePort, a.runtimeConfig.AllowedOrigins, pipeline)
	if err != nil {
		log.Fatalf("failed to start HTTP server: %s", err)
	}
  ......
}
```

startHTTPServer() 的详细代码:

```go
func (a *DaprRuntime) startHTTPServer(port int, publicPort *int, profilePort int, allowedOrigins string, pipeline http_middleware.Pipeline) error {
	a.daprHTTPAPI = http.NewAPI(a.runtimeConfig.ID, a.appChannel, a.directMessaging, a.getComponents, a.stateStores, a.secretStores,
		a.secretsConfiguration, a.getPublishAdapter(), a.actor, a.sendToOutputBinding, a.globalConfig.Spec.TracingSpec, a.ShutdownWithWait)
	serverConf := http.NewServerConfig(a.runtimeConfig.ID, a.hostAddress, port, a.runtimeConfig.APIListenAddresses, publicPort, profilePort, allowedOrigins, a.runtimeConfig.EnableProfiling, a.runtimeConfig.MaxRequestBodySize, a.runtimeConfig.UnixDomainSocket, a.runtimeConfig.ReadBufferSize, a.runtimeConfig.StreamRequestBody, a.runtimeConfig.APILogLevel)

	server := http.NewServer(a.daprHTTPAPI, serverConf, a.globalConfig.Spec.TracingSpec, a.globalConfig.Spec.MetricSpec, pipeline, a.globalConfig.Spec.APISpec)
	if err := server.StartNonBlocking(); err != nil {
		return err
	}
	a.apiClosers = append(a.apiClosers, server)

	return nil
}
```





StartNonBlocking() 的实现代码在 `pkg/http/server.go` 中：

```go
// StartNonBlocking starts a new server in a goroutine.
func (s *server) StartNonBlocking() error {
  	......
  	for _, apiListenAddress := range s.config.APIListenAddresses {
			l, err := net.Listen("tcp", fmt.Sprintf("%s:%v", apiListenAddress, s.config.Port))
      listeners = append(listeners, l)
		}
  
  	for _, listener := range listeners {
		// customServer is created in a loop because each instance
		// has a handle on the underlying listener.
		customServer := &fasthttp.Server{
			Handler:            handler,
			MaxRequestBodySize: s.config.MaxRequestBodySize * 1024 * 1024,
			ReadBufferSize:     s.config.ReadBufferSize * 1024,
			StreamRequestBody:  s.config.StreamRequestBody,
		}
		s.servers = append(s.servers, customServer)

		go func(l net.Listener) {
			if err := customServer.Serve(l); err != nil {
				log.Fatal(err)
			}
		}(listener)
	}
}
```

