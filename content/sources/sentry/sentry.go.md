---
title: "sentry代码"
linkTitle: "sentry.go"
weight: 30
date: 2022-02-22
description: >
  
---



sentry 模块的主要实现在文件 `pkg/sentry/sentry.go` 中。



## 定义



### 定义 CA 接口

```go
type CertificateAuthority interface {
	Start(context.Context, config.SentryConfig) error
	Stop()
	Restart(context.Context, config.SentryConfig) error
}
```

start 和 restart 的函数定义是一样的。

### 定义 sentry 结构体

```go
type sentry struct {
	conf        config.SentryConfig    // sentry的配置，启动时由 main 函数初始化后传入
	ctx         context.Context				 // 启动时由 main 函数初始化后传入
	cancel      context.CancelFunc     
	server      server.CAServer        // CA server
	restartLock sync.Mutex             // 用于 restart 的锁
	running     chan bool
	stopping    chan bool
}
```



## 主流程

Sentry.go 被 sentry main.go 调用，主要工作流程就是三个事情：

```go
// 1. 初始化
ca := sentry.NewSentryCA()
// 2. 启动
err = ca.Start(runCtx, config)
// 3. 在需要时重启
innerErr := ca.Restart(runCtx, config)
```

备注：sentry main.go 没有调用 sentry的 stop()，这个 stop() 只在 restart() 方法中被调用。

### 初始化 sentry

NewSentryCA() 的实现：

```go
// NewSentryCA returns a new Sentry Certificate Authority instance.
func NewSentryCA() CertificateAuthority {
	return &sentry{
		running: make(chan bool, 1),
	}
}
```

什么都没干，只是初始化了 running 这个channel。



### 启动 sentry

```go
// Start the server in background.
func (s *sentry) Start(ctx context.Context, conf config.SentryConfig) error {
	// If the server is already running, return an error
	select {
	case s.running <- true:
	default:
		return errors.New("CertificateAuthority server is already running")
	}

	// Create the CA server
	s.conf = conf
	certAuth, v := s.createCAServer()

	// Start the server in background
	s.ctx, s.cancel = context.WithCancel(ctx)
	go s.run(certAuth, v)

	// Wait 100ms to ensure a clean startup
	time.Sleep(100 * time.Millisecond)

	return nil
}
```

主要工作就是创建 CA server，然后运行服务。

#### 创建 ca server

createCAServer() 方法加载信任锚和签发者证书，然后创建一个新的CA：

```go
// Loads the trust anchors and issuer certs, then creates a new CA.
func (s *sentry) createCAServer() (ca.CertificateAuthority, identity.Validator) {
	// Create CA
	certAuth, authorityErr := ca.NewCertificateAuthority(s.conf)
	if authorityErr != nil {
		log.Fatalf("error getting certificate authority: %s", authorityErr)
	}
	log.Info("certificate authority loaded")

	// Load the trust bundle
	trustStoreErr := certAuth.LoadOrStoreTrustBundle()
	if trustStoreErr != nil {
		log.Fatalf("error loading trust root bundle: %s", trustStoreErr)
	}
	certExpiry := certAuth.GetCACertBundle().GetIssuerCertExpiry()
	if certExpiry == nil {
		log.Fatalf("error loading trust root bundle: missing certificate expiry")
	} else {
		// Need to be in an else block for the linter
		log.Infof("trust root bundle loaded. issuer cert expiry: %s", certExpiry.String())
	}
	monitoring.IssuerCertExpiry(certExpiry)

	// Create identity validator
	v, validatorErr := s.createValidator()
	if validatorErr != nil {
		log.Fatalf("error creating validator: %s", validatorErr)
	}
	log.Info("validator created")

	return certAuth, v
}
```

方法返回 ca.CertificateAuthority 和 identity.Validator 。

#### 创建 identity.Validator

createValidator 的实现细节：

```go
func (s *sentry) createValidator() (identity.Validator, error) {
	if config.IsKubernetesHosted() {  // 通过 KUBERNETES_SERVICE_HOST 环境变量来判断
		// we're in Kubernetes, create client and init a new serviceaccount token validator
		kubeClient, err := k8s.GetClient()
		if err != nil {
			return nil, fmt.Errorf("failed to create kubernetes client: %w", err)
		}

		// TODO: Remove once the NoDefaultTokenAudience feature is finalized
		noDefaultTokenAudience := false

    // 创建 kubernetes 的 Validator
		return kubernetes.NewValidator(kubeClient, s.conf.GetTokenAudiences(), noDefaultTokenAudience), nil
	}
  
  // 创建 selfhosted 的 Validator
	return selfhosted.NewValidator(), nil
}
```

#### 运行 sentry

run 方法运行 CA server，阻塞直到服务器关闭：

```go
// Runs the CA server.
// This method blocks until the server is shut down.
func (s *sentry) run(certAuth ca.CertificateAuthority, v identity.Validator) {
	s.server = server.NewCAServer(certAuth, v)

	// In background, watch for the root certificate's expiration
	go watchCertExpiry(s.ctx, certAuth)

	// Watch for context cancelation to stop the server
	go func() {
		<-s.ctx.Done()
		s.server.Shutdown()
		close(s.running)
		s.running = make(chan bool, 1)
		if s.stopping != nil {
			close(s.stopping)
		}
	}()

	// Start the server; this is a blocking call
	log.Infof("sentry certificate authority is running, protecting y'all")
	serverRunErr := s.server.Run(s.conf.Port, certAuth.GetCACertBundle())
	if serverRunErr != nil {
		log.Fatalf("error starting gRPC server: %s", serverRunErr)
	}
}
```

启动 ca 的 grpc server 以便接收外部请求。

#### 监控证书过期

Run() 方法中启动了一个 goroutine，用于监控证书是否过期。如果快要过期了，则会显示错误信息。

```go
// Watches certificates' expiry and shows an error message when they're nearing expiration time.
// This is a blocking method that should be run in its own goroutine.
func watchCertExpiry(ctx context.Context, certAuth ca.CertificateAuthority) {
	log.Debug("starting root certificate expiration watcher")
  // time 是每小时触发一次
	certExpiryCheckTicker := time.NewTicker(time.Hour)
	for {
		select {
		case <-certExpiryCheckTicker.C:
			caCrt := certAuth.GetCACertBundle().GetRootCertPem()
			block, _ := pem.Decode(caCrt)
			cert, certParseErr := x509.ParseCertificate(block.Bytes)
			if certParseErr != nil {
				log.Warn("could not determine Dapr root certificate expiration time")
				break
			}
			if cert.NotAfter.Before(time.Now().UTC()) {
        // 已经过期则报警
				log.Warn("Dapr root certificate expiration warning: certificate has expired.")
				break
			}
			if (cert.NotAfter.Add(-30 * 24 * time.Hour)).Before(time.Now().UTC()) {
        // 有效期不足30天也报警
				expiryDurationHours := int(cert.NotAfter.Sub(time.Now().UTC()).Hours())
				log.Warnf("Dapr root certificate expiration warning: certificate expires in %d days and %d hours", expiryDurationHours/24, expiryDurationHours%24)
			} else {
				validity := cert.NotAfter.Sub(time.Now().UTC())
				log.Debugf("Dapr root certificate is still valid for %s", validity.String())
			}
		case <-ctx.Done():
			log.Debug("terminating root certificate expiration watcher")
			certExpiryCheckTicker.Stop()
			return
		}
	}
}
```



### 停止 sentry 



```go
// Stop the server.
func (s *sentry) Stop() {
	log.Info("sentry certificate authority is shutting down")
	if s.cancel != nil {
		s.stopping = make(chan bool)
		s.cancel()
		<-s.stopping
		s.stopping = nil
	}
}
```



### 重启 sentry

Restart() 方法重启 sentry：

```go
func (s *sentry) Restart(ctx context.Context, conf config.SentryConfig) error {
	s.restartLock.Lock()
	defer s.restartLock.Unlock()
	log.Info("sentry certificate authority is restarting")
	s.Stop()
	// Wait 200ms to ensure a clean shutdown
	time.Sleep(200 * time.Millisecond)
	return s.Start(ctx, conf)
}
```

步骤：

- 先加锁
- 停止 sentry
- sleep 200 毫秒
- 再启动 sentry
