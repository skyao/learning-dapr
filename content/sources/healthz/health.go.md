---
type: docs
title: "health.go的源码学习"
linkTitle: "health.go"
weight: 8001
date: 2021-03-09
description: >
  health checking的客户端实现
---

Dapr health package中的 health.go 文件的源码分析，health checking的客户端实现

## 代码实现

### Option 方法定义

```go
// Option is an a function that applies a health check option
type Option func(o *healthCheckOptions)
```

### healthCheckOptions 结构体定义

healthCheckOptions 结构体

```go
type healthCheckOptions struct {
	initialDelay      time.Duration
	requestTimeout    time.Duration
	failureThreshold  int
	interval          time.Duration
	successStatusCode int
}
```

### With系列方法

WithXxx 方法用来设置上述5个健康检查的选项，每个方法都返回一个 Option 函数：

```go
// WithInitialDelay sets the initial delay for the health check
func WithInitialDelay(delay time.Duration) Option {
	return func(o *healthCheckOptions) {
		o.initialDelay = delay
	}
}

// WithFailureThreshold sets the failure threshold for the health check
func WithFailureThreshold(threshold int) Option {
	return func(o *healthCheckOptions) {
		o.failureThreshold = threshold
	}
}

// WithRequestTimeout sets the request timeout for the health check
func WithRequestTimeout(timeout time.Duration) Option {
	return func(o *healthCheckOptions) {
		o.requestTimeout = timeout
	}
}

// WithSuccessStatusCode sets the status code for the health check
func WithSuccessStatusCode(code int) Option {
	return func(o *healthCheckOptions) {
		o.successStatusCode = code
	}
}

// WithInterval sets the interval for the health check
func WithInterval(interval time.Duration) Option {
	return func(o *healthCheckOptions) {
		o.interval = interval
	}
}
```

### StartEndpointHealthCheck 方法

StartEndpointHealthCheck 方法用给定的选项在指定的地址上启动健康检查。它返回一个通道，如果端点是健康的则发出true，如果满足失败条件则发出false。

```go
// StartEndpointHealthCheck starts a health check on the specified address with the given options.
// It returns a channel that will emit true if the endpoint is healthy and false if the failure conditions
// Have been met.
func StartEndpointHealthCheck(endpointAddress string, opts ...Option) chan bool {
	options := &healthCheckOptions{}
	applyDefaults(options)

   // 执行每个 Option 函数来设置健康检查的选项
	for _, o := range opts {
		o(options)
	}
	signalChan := make(chan bool, 1)

	go func(ch chan<- bool, endpointAddress string, options *healthCheckOptions) {
      // 设置健康检查的间隔时间 interval，默认5秒一次
		ticker := time.NewTicker(options.interval)
		failureCount := 0
      // 先 sleep initialDelay 时间再开始健康检查
		time.Sleep(options.initialDelay)

      // 创建 http client，设置请求超时时间为 requestTimeout
		client := &fasthttp.Client{
			MaxConnsPerHost:           5, // Limit Keep-Alive connections
			ReadTimeout:               options.requestTimeout,
			MaxIdemponentCallAttempts: 1,
		}

		req := fasthttp.AcquireRequest()
		req.SetRequestURI(endpointAddress)
		req.Header.SetMethod(fasthttp.MethodGet)
		defer fasthttp.ReleaseRequest(req)

		for range ticker.C {
			resp := fasthttp.AcquireResponse()
			err := client.DoTimeout(req, resp, options.requestTimeout)
         // 通过检查应答的状态码来判断健康检查是否成功： successStatusCode
			if err != nil || resp.StatusCode() != options.successStatusCode {
            // 健康检查失败，错误计数器加一
				failureCount++
            // 如果连续错误次数达到阈值 failureThreshold，则视为健康检查失败，发送false到channel
				if failureCount == options.failureThreshold {
					ch <- false
				}
			} else {
            // 健康检查成功，发送 true 到 channel
				ch <- true
            // 同时重制 failureCount
				failureCount = 0
			}
			fasthttp.ReleaseResponse(resp)
		}
	}(signalChan, endpointAddress, options)
	return signalChan
}
```

applyDefaults() 方法设置默认属性：

```go
const (
	initialDelay      = time.Second * 1
	failureThreshold  = 2
	requestTimeout    = time.Second * 2
	interval          = time.Second * 5
	successStatusCode = 200
)

func applyDefaults(o *healthCheckOptions) {
   o.failureThreshold = failureThreshold
   o.initialDelay = initialDelay
   o.requestTimeout = requestTimeout
   o.successStatusCode = successStatusCode
   o.interval = interval
}
```

## 健康检查方式总结

对某一个给定地址 endpointAddress 进行健康检查的步骤和方式为：

1. 先 sleep initialDelay 时间再开始健康检查：可能对方还在初始化过程中
2. 每隔间隔时间 interval 时间发起一次健康检查
3. 每次健康检查是向目标地址 endpointAddress 发起一个 HTTP GET 请求，超时时间为 requestTimeout
4. 检查应答判断是否健康
   - 返回应答并且应答的状态码是 successStatusCode 则视为本地健康检查成功
   - 超时或者应答的状态码不是 successStatusCode 则视为本地健康检查失败
5. 如果失败则开始累加计数器，然后间隔 interval 时间之后再次进行健康检查
6. 如果多次失败，累计达到阈值 failureThreshold，报告为健康检查失败
7. 只要单次成功，则清理之前的错误累计次数，报告为健康检查成功。