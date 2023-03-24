---
title: "CA server代码"
linkTitle: "server.go"
weight: 40
date: 2022-02-22
description: >
  
---



ca server 的实现在文件 `pkg/sentry/server/server.go` 中。

## 定义

### CAServer 接口

```go
// CAServer is an interface for the Certificate Authority server.
type CAServer interface {
	Run(port int, trustBundle ca.TrustRootBundler) error
	Shutdown()
}
```

### server 结构体

```go
type server struct {
	certificate *tls.Certificate
	certAuth    ca.CertificateAuthority
	srv         *grpc.Server    // grpc server，用来对外提供 grpc 服务
	validator   identity.Validator
}
```





## 主流程

server.go 被 sentry.go 调用，主要工作流程就是三个事情：

```go
// 1. 初始化CA server
s.server = server.NewCAServer(certAuth, v)
// 2. 运行CA server
s.server.Run(s.conf.Port, certAuth.GetCACertBundle())
// 3. 在需要时关闭CA server
s.server.Shutdown()
```



### 初始化CA server

```go
// NewCAServer returns a new CA Server running a gRPC server.
func NewCAServer(ca ca.CertificateAuthority, validator identity.Validator) CAServer {
	return &server{
		certAuth:  ca,
		validator: validator,
	}
}
```

保存传递进来的参数，这两个参数在 sentry.go 中初始化。

### 运行 CA server

CA server 主要提供两个功能：

1. 基本的 grpc 服务：以便为客户端提供服务
2. 安全：必须为提供的服务进行安全保护，因此客户端必须实用 trust root cert

```go
// Run starts a secured gRPC server for the Sentry Certificate Authority.
// It enforces client side cert validation using the trust root cert.
func (s *server) Run(port int, trustBundler ca.TrustRootBundler) error {
	addr := fmt.Sprintf(":%d", port)
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		return fmt.Errorf("could not listen on %s: %w", addr, err)
	}

	tlsOpt := s.tlsServerOption(trustBundler)
  // 创建 grpc server
	s.srv = grpc.NewServer(tlsOpt)
  // 注册 ca server 到 grpc server
	sentryv1pb.RegisterCAServer(s.srv, s)

  // 启动 grpc server 监听服务地址
	if err := s.srv.Serve(lis); err != nil {
		return fmt.Errorf("grpc serve error: %w", err)
	}
	return nil
}
```

trustBundler 是从 sentry.go 中传递过来，后面详细展开。

### 关闭 CA server

```go
func (s *server) Shutdown() {
	if s.srv != nil {
    // 调用 grpc 的 GracefulStop，会在请求完成后再关闭
		s.srv.GracefulStop()
	}
}
```



## 客户端安全

tlsServerOption() 方法，为客户端连接准备 tls 相关的选项：

```go
func (s *server) tlsServerOption(trustBundler ca.TrustRootBundler) grpc.ServerOption {
	cp := trustBundler.GetTrustAnchors()

	//nolint:gosec
	config := &tls.Config{
		ClientCAs: cp,
    // 这里要求验证客户端证书
		// Require cert verification
		ClientAuth: tls.RequireAndVerifyClientCert,
		GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
			if s.certificate == nil || needsRefresh(s.certificate, serverCertExpiryBuffer) {
        // 如果ca server的证书为空，或者需要刷新，则开始创建/刷新证书
				cert, err := s.getServerCertificate()
				if err != nil {
					monitoring.ServerCertIssueFailed("server_cert")
					log.Error(err)
					return nil, fmt.Errorf("failed to get TLS server certificate: %w", err)
				}
				s.certificate = cert
			}
			return s.certificate, nil
		},
	}
	return grpc.Creds(credentials.NewTLS(config))
}
```

needsRefresh() 方法的实现：

```go
func needsRefresh(cert *tls.Certificate, expiryBuffer time.Duration) bool {
	leaf := cert.Leaf
	if leaf == nil {
		return true
	}

	// Check if the leaf certificate is about to expire.
  // 检查是不是快要过期了:15 分钟
	return leaf.NotAfter.Add(-serverCertExpiryBuffer).Before(time.Now().UTC())
}
const (
	serverCertExpiryBuffer = time.Minute * 15
)
```

getServerCertificate() 方法负责生成服务器端的证书：

```go
func (s *server) getServerCertificate() (*tls.Certificate, error) {
	csrPem, pkPem, err := csr.GenerateCSR("", false)
	if err != nil {
		return nil, err
	}

	now := time.Now().UTC()
	issuerExp := s.certAuth.GetCACertBundle().GetIssuerCertExpiry()
	if issuerExp == nil {
		return nil, errors.New("could not find expiration in issuer certificate")
	}
	serverCertTTL := issuerExp.Sub(now)

	resp, err := s.certAuth.SignCSR(csrPem, s.certAuth.GetCACertBundle().GetTrustDomain(), nil, serverCertTTL, false)
	if err != nil {
		return nil, err
	}

	certPem := resp.CertPEM
	certPem = append(certPem, s.certAuth.GetCACertBundle().GetIssuerCertPem()...)
	if rootCertPem := s.certAuth.GetCACertBundle().GetRootCertPem(); len(rootCertPem) > 0 {
		certPem = append(certPem, rootCertPem...)
	}

	cert, err := tls.X509KeyPair(certPem, pkPem)
	if err != nil {
		return nil, err
	}

	return &cert, nil
}
```

更多细节要看 certAuth.SignCSR() 方法的实现。



## 签署证书

SignCertificate() 方法处理从 dapr sidedar 发起的 CSR 请求。这个方法接收带有 identity 和 初始证书的请求，并为调用者返回包括信任链在内的签名证书和过期时间。

```go
// SignCertificate handles CSR requests originating from Dapr sidecars.
// The method receives a request with an identity and initial cert and returns
// A signed certificate including the trust chain to the caller along with an expiry date.
func (s *server) SignCertificate(ctx context.Context, req *sentryv1pb.SignCertificateRequest) (*sentryv1pb.SignCertificateResponse, error) {
	monitoring.CertSignRequestReceived()

	csrPem := req.GetCertificateSigningRequest()

  // 解析请求中的 CSR
	csr, err := certs.ParsePemCSR(csrPem)
	if err != nil {
		err = fmt.Errorf("cannot parse certificate signing request pem: %w", err)
		log.Error(err)
		monitoring.CertSignFailed("cert_parse")
		return nil, err
	}

  // 验证 CSR
	err = s.certAuth.ValidateCSR(csr)
	if err != nil {
		err = fmt.Errorf("error validating csr: %w", err)
		log.Error(err)
		monitoring.CertSignFailed("cert_validation")
		return nil, err
	}

  // 验证请求身份
	err = s.validator.Validate(req.GetId(), req.GetToken(), req.GetNamespace())
	if err != nil {
		err = fmt.Errorf("error validating requester identity: %w", err)
		log.Error(err)
		monitoring.CertSignFailed("req_id_validation")
		return nil, err
	}

  // 签名证书
	identity := identity.NewBundle(csr.Subject.CommonName, req.GetNamespace(), req.GetTrustDomain())
	signed, err := s.certAuth.SignCSR(csrPem, csr.Subject.CommonName, identity, -1, false)
	if err != nil {
		err = fmt.Errorf("error signing csr: %w", err)
		log.Error(err)
		monitoring.CertSignFailed("cert_sign")
		return nil, err
	}

  // 准备要返回的各种数据
	certPem := signed.CertPEM
	issuerCert := s.certAuth.GetCACertBundle().GetIssuerCertPem()
	rootCert := s.certAuth.GetCACertBundle().GetRootCertPem()

	certPem = append(certPem, issuerCert...)
	if len(rootCert) > 0 {
		certPem = append(certPem, rootCert...)
	}

	if len(certPem) == 0 {
		err = errors.New("insufficient data in certificate signing request, no certs signed")
		log.Error(err)
		monitoring.CertSignFailed("insufficient_data")
		return nil, err
	}

	expiry := timestamppb.New(signed.Certificate.NotAfter)
	if err = expiry.CheckValid(); err != nil {
		return nil, fmt.Errorf("could not validate certificate validity: %w", err)
	}

  // 组装 response 结构体
	resp := &sentryv1pb.SignCertificateResponse{
		WorkloadCertificate:    certPem,
		TrustChainCertificates: [][]byte{issuerCert, rootCert},
		ValidUntil:             expiry,
	}

	monitoring.CertSignSucceed()

	return resp, nil
}
```



## 总结

实现很简单，就是涉及到证书的各种操作，需要有相关的背景知识。

