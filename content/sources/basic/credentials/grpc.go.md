---
type: docs
title: "grpc.go的源码学习"
linkTitle: "grpc.go"
weight: 365
date: 2021-03-12
description: >
  获取服务器端选项和客户端选项
---

Dapr credentials package中的 grpc.go文件的源码学习，获取服务器端选项和客户端选项。

### GetServerOptions() 方法


```go
func GetServerOptions(certChain *CertChain) ([]grpc.ServerOption, error) {
	opts := []grpc.ServerOption{}
	if certChain == nil {
		return opts, nil
	}

	cp := x509.NewCertPool()
	cp.AppendCertsFromPEM(certChain.RootCA)

	cert, err := tls.X509KeyPair(certChain.Cert, certChain.Key)
	if err != nil {
		return opts, nil
	}

	// nolint:gosec
	config := &tls.Config{
		ClientCAs: cp,
		// Require cert verification
		ClientAuth:   tls.RequireAndVerifyClientCert,
		Certificates: []tls.Certificate{cert},
	}
	opts = append(opts, grpc.Creds(credentials.NewTLS(config)))

	return opts, nil
}
```

### GetClientOptions() 方法


```go
func GetClientOptions(certChain *CertChain, serverName string) ([]grpc.DialOption, error) {
	opts := []grpc.DialOption{}
	if certChain != nil {
		cp := x509.NewCertPool()
		ok := cp.AppendCertsFromPEM(certChain.RootCA)
		if !ok {
			return nil, errors.New("failed to append PEM root cert to x509 CertPool")
		}
		config, err := TLSConfigFromCertAndKey(certChain.Cert, certChain.Key, serverName, cp)
		if err != nil {
			return nil, errors.Wrap(err, "failed to create tls config from cert and key")
		}
		opts = append(opts, grpc.WithTransportCredentials(credentials.NewTLS(config)))
	} else {
		opts = append(opts, grpc.WithInsecure())
	}
	return opts, nil
}
```

> TODO: 好吧，细节后面看，加密我不熟。

