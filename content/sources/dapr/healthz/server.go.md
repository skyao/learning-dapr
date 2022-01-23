---
type: docs
title: "server.go的源码学习"
linkTitle: "server.go"
weight: 8002
date: 2021-03-09
description: >
  healthz server的实现
---

Dapr health package中的 server.go 文件的源码分析，healthz server的实现

## 代码实现

### Health server

healthz server 的接口定义：

```go
// Server is the interface for the healthz server
type Server interface {
	Run(context.Context, int) error
	Ready()
	NotReady()
}
```

server 结构体，ready 字段保存状态：

```go
type server struct {
	ready bool
	log   logger.Logger
}
```

创建 healthz server的方法：

```go
// NewServer returns a new healthz server
func NewServer(log logger.Logger) Server {
   return &server{
      log: log,
   }
}
```

设置 ready 状态的两个方法：

```go
// Ready sets a ready state for the endpoint handlers
func (s *server) Ready() {
   s.ready = true
}

// NotReady sets a not ready state for the endpoint handlers
func (s *server) NotReady() {
   s.ready = false
}
```



### 运行healthz server

Run 方法启动一个带有 healthz 端点的 http 服务器，端口通过参数 port 指定：

```go
// Run starts a net/http server with a healthz endpoint
func (s *server) Run(ctx context.Context, port int) error {
   router := http.NewServeMux()
   router.Handle("/healthz", s.healthz())

   srv := &http.Server{
      Addr:    fmt.Sprintf(":%d", port),
      Handler: router,
   }
   ...
}
```

启动之后：

```go
   doneCh := make(chan struct{})

   go func() {
      select {
      case <-ctx.Done():
         s.log.Info("Healthz server is shutting down")
         shutdownCtx, cancel := context.WithTimeout(
            context.Background(),
            time.Second*5,
         )
         defer cancel()
         srv.Shutdown(shutdownCtx) // nolint: errcheck
      case <-doneCh:
      }
   }()

   s.log.Infof("Healthz server is listening on %s", srv.Addr)
   err := srv.ListenAndServe()
   if err != http.ErrServerClosed {
      s.log.Errorf("Healthz server error: %s", err)
   }
   close(doneCh)
   return err
}
```

### healthz server 处理请求

healthz() 方法是 health endpoint 的 handler，根据当前 healthz server 的 ready 字段的状态值返回 HTTP 状态码：

```go
// healthz is a health endpoint handler
func (s *server) healthz() http.Handler {
   return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      var status int
      if s.ready {
      	// ready 返回 200
         status = http.StatusOK
      } else {
         // 不 ready 则返回 503
         status = http.StatusServiceUnavailable
      }
      w.WriteHeader(status)
   })
}
```

## 使用场景

healthz server 在 injector / placement / sentry / operator 中都有使用，这些进程都是在 main 方法中启动 healthz server。

### injector

injector 启动在 8080 端口：

```go
const (
	healthzPort = 8080
)

func main() {
   ......
	go func() {
		healthzServer := health.NewServer(log)
		healthzServer.Ready()

		healthzErr := healthzServer.Run(ctx, healthzPort)
		if healthzErr != nil {
			log.Fatalf("failed to start healthz server: %s", healthzErr)
		}
	}()
	......
}
```

### placement

placement 默认启动在 8080 端口（也可以通过命令行参数修改端口）：

```go
const (
	defaultHealthzPort       = 8080
)

func main() {
	flag.IntVar(&cfg.healthzPort, "healthz-port", cfg.healthzPort, "sets the HTTP port for the healthz server")
   ......
	go startHealthzServer(cfg.healthzPort)
	......
}

func startHealthzServer(healthzPort int) {
	healthzServer := health.NewServer(log)
	healthzServer.Ready()

	if err := healthzServer.Run(context.Background(), healthzPort); err != nil {
		log.Fatalf("failed to start healthz server: %s", err)
	}
}
```

### sentry

sentry 启动在 8080 端口：

```go
const (
	healthzPort = 8080
)

func main() {
   ......
	go func() {
		healthzServer := health.NewServer(log)
		healthzServer.Ready()

		err := healthzServer.Run(ctx, healthzPort)
		if err != nil {
			log.Fatalf("failed to start healthz server: %s", err)
		}
	}()
	......
}
```

### operator

operator 启动在 8080 端口：

```go
const (
	healthzPort = 8080
)

func main() {
   ......
	go func() {
		healthzServer := health.NewServer(log)
		healthzServer.Ready()

		err := healthzServer.Run(ctx, healthzPort)
		if err != nil {
			log.Fatalf("failed to start healthz server: %s", err)
		}
	}()
	......
}
```

### darpd

特别指出：daprd 没有使用 healthz server，daprd 是直接在 dapr HTTP api 的基础上增加了 healthz 的功能。

具体代码在 http/api.go 中：

```go
func NewAPI(......
   api.endpoints = append(api.endpoints, api.constructHealthzEndpoints()...)
	return api
}

func (a *api) constructHealthzEndpoints() []Endpoint {
   return []Endpoint{
      {
         Methods: []string{fasthttp.MethodGet},
         Route:   "healthz",
         Version: apiVersionV1,
         Handler: a.onGetHealthz,
      },
   }
}
```

onGetHealthz() 方法处理请求：

```go
func (a *api) onGetHealthz(reqCtx *fasthttp.RequestCtx) {
   if !a.readyStatus {
      msg := NewErrorResponse("ERR_HEALTH_NOT_READY", messages.ErrHealthNotReady)
      respondWithError(reqCtx, fasthttp.StatusInternalServerError, msg)
      log.Debug(msg)
   } else {
      respondEmpty(reqCtx)
   }
}

func respondEmpty(ctx *fasthttp.RequestCtx) {
	ctx.Response.SetBody(nil)
	ctx.Response.SetStatusCode(fasthttp.StatusNoContent)
}
```

注意：这里成功时返回的状态码是 204 StatusNoContent，而不是通常的 200 OK。