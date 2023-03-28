---
title: "metrics代码"
linkTitle: "metrics.go"
weight: 70
date: 2022-02-22
description: >
  sentry 中的 metrics 实现
---



metrics 相关的实现在文件 `pkg/sentry/monitoring/metrics.go` 中。



## 准备工作

### 变量定义

定义了一些和 metrics 相关的变量：

```go
var (
	// Metrics definitions.
	csrReceivedTotal = stats.Int64(
		"sentry/cert/sign/request_received_total",
		"The number of CSRs received.",
		stats.UnitDimensionless)
	certSignSuccessTotal = stats.Int64(
		"sentry/cert/sign/success_total",
		"The number of certificates issuances that have succeeded.",
		stats.UnitDimensionless)
	certSignFailedTotal = stats.Int64(
		"sentry/cert/sign/failure_total",
		"The number of errors occurred when signing the CSR.",
		stats.UnitDimensionless)
	serverTLSCertIssueFailedTotal = stats.Int64(
		"sentry/servercert/issue_failed_total",
		"The number of server TLS certificate issuance failures.",
		stats.UnitDimensionless)
	issuerCertChangedTotal = stats.Int64(
		"sentry/issuercert/changed_total",
		"The number of issuer cert updates, when issuer cert or key is changed",
		stats.UnitDimensionless)
	issuerCertExpiryTimestamp = stats.Int64(
		"sentry/issuercert/expiry_timestamp",
		"The unix timestamp, in seconds, when issuer/root cert will expire.",
		stats.UnitDimensionless)

	// Metrics Tags.
	failedReasonKey = tag.MustNewKey("reason")
	noKeys          = []tag.Key{}
)
```

目前总共有 6 个 metrics 指标：

- csrReceivedTotal：接收到的 csr 的数量
- certSignSuccessTotal：签署成功的证书数量
- certSignFailedTotal：签署失败的证书数量
- serverTLSCertIssueFailedTotal：服务器TLS证书发放失败的次数。
- issuerCertChangedTotal： 当签发人的证书或钥匙被改变时，签发人证书更新的数量
- issuerCertExpiryTimestamp：发行人/根证书有效期的unix时间戳，单位是秒。



## 初始化

初始化 metrics：

```go
func InitMetrics() error {
  // 将 6 个 metrics 指标都注册起来
	return view.Register(
		diagUtils.NewMeasureView(csrReceivedTotal, noKeys, view.Count()),
		diagUtils.NewMeasureView(certSignSuccessTotal, noKeys, view.Count()),
		diagUtils.NewMeasureView(certSignFailedTotal, []tag.Key{failedReasonKey}, view.Count()),
		diagUtils.NewMeasureView(serverTLSCertIssueFailedTotal, []tag.Key{failedReasonKey}, view.Count()),
		diagUtils.NewMeasureView(issuerCertChangedTotal, noKeys, view.Count()),
		diagUtils.NewMeasureView(issuerCertExpiryTimestamp, noKeys, view.LastValue()),
	)
}
```



## 收集 metrics

### crs 相关

CertSignRequestReceived() 对接收到的 csr 数量进行计数：

```go
// CertSignRequestReceived counts when CSR received.
func CertSignRequestReceived() {
	stats.Record(context.Background(), csrReceivedTotal.M(1))
}
```

另外 CertSignSucceed() 会对处理成功的情况进行计数：

```go
func CertSignSucceed() {
	stats.Record(context.Background(), certSignSuccessTotal.M(1))
}
```

而 CertSignFailed() 则会对处理失败的情况进行计数：

```go
func CertSignFailed(reason string) {
	stats.RecordWithTags(
		context.Background(),
		diagUtils.WithTags(certSignFailedTotal.Name(), failedReasonKey, reason),
		certSignFailedTotal.M(1))
}
```

三者的调用点为 server.go 中的 SignCertificate() 函数，这个函数负责处理 csr 请求：

```go
func (s *server) SignCertificate(ctx context.Context, req *sentryv1pb.SignCertificateRequest) (*sentryv1pb.SignCertificateResponse, error) {
  // 进来就计数：这是 接收到的 csr 数量
	monitoring.CertSignRequestReceived()
  ......
  
  // 每一个错误在return之前都要进行一次失败计数
	if err != nil {
		monitoring.CertSignFailed("cert_parse")
		return nil, err
	}
  ......
  // 如果最后 csr 处理成功，则进行成功计数
  monitoring.CertSignSucceed()

	return resp, nil
}
```



### 证书有效期

IssuerCertExpiry() 方法记录 root cert 有效期的情况：

```go
// IssuerCertExpiry records root cert expiry.
func IssuerCertExpiry(expiry *time.Time) {
	stats.Record(context.Background(), issuerCertExpiryTimestamp.M(expiry.Unix()))
}
```

调用点在 sentry.go 中的 createCAServer() 函数中：

```go
func (s *sentry) createCAServer(ctx context.Context) (ca.CertificateAuthority, identity.Validator) {
	certAuth, authorityErr := ca.NewCertificateAuthority(s.conf)
	trustStoreErr := certAuth.LoadOrStoreTrustBundle(ctx)
	......
	certExpiry := certAuth.GetCACertBundle().GetIssuerCertExpiry()
	monitoring.IssuerCertExpiry(certExpiry)
	......
	return certAuth, v
}
```

在 CA server 的创建过程中，会加载 trust bundle并检查证书的有效期，在这里记录有效期的数据收集。



### 服务器证书签发失败

ServerCertIssueFailed() 记录服务器证书签发失败。

```go
func ServerCertIssueFailed(reason string) {
	stats.Record(context.Background(), serverTLSCertIssueFailedTotal.M(1))
}
```

调用点在 server.go 中：

```go

func (s *server) Run(ctx context.Context, port int, trustBundler ca.TrustRootBundler) error {
  ......
  tlsOpt := s.tlsServerOption(trustBundler)
  s.srv = grpc.NewServer(tlsOpt)
  ......
}
```

sentry server启动过程中，在启动 grpc server 时，需要获取 tls server 的参数，期间要获取 sentry server 的服务器端证书：

```go
func (s *server) tlsServerOption(trustBundler ca.TrustRootBundler) grpc.ServerOption {
	cp := trustBundler.GetTrustAnchors()

	config := &tls.Config{
		ClientCAs: cp,
		// Require cert verification
		ClientAuth: tls.RequireAndVerifyClientCert,
		GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
			if s.certificate == nil || needsRefresh(s.certificate, serverCertExpiryBuffer) {
				cert, err := s.getServerCertificate()
				if err != nil {
					monitoring.ServerCertIssueFailed("server_cert")
					log.Error(err)
					return nil, fmt.Errorf("failed to get TLS server certificate: %w", err)
				}
				s.certificate = cert
			}
	......
}
```

如果获取失败，则会记录这个失败信息。

### 发行者证书变更

IssuerCertChanged() 记录发行人凭证的变更：

```go
func IssuerCertChanged() {
	stats.Record(context.Background(), issuerCertChangedTotal.M(1))
}
```

调用点在 main.go 中的 main() 函数中，sentry 在启动后会监视发行者证书（默认为 "/var/run/dapr/credentials" 下的  "issuer.crt" 文件）：

```go
func main() {
  ......
			func(ctx context.Context) error {
				select {
				case <-ctx.Done():
					return nil

				case <-issuerEvent:
					monitoring.IssuerCertChanged()
					log.Debug("received issuer credentials changed signal")
				......
	}
  ......
  	// Watch for changes in the watchDir
	mngr.Add(func(ctx context.Context) error {
		log.Infof("starting watch on filesystem directory: %s", watchDir)
		return fswatcher.Watch(ctx, watchDir, issuerEvent)
	})
}
```

