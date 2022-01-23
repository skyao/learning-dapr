---
type: docs
title: "tls.go的源码学习"
linkTitle: "tls.go"
weight: 363
date: 2021-03-12
description: >
  从 cert/key 中装载 tls.config 对象
---

Dapr credentials package中的 tls.go文件的源码学习，从 cert/key 中装载 tls.config 对象。

### TLSConfigFromCertAndKey() 方法

TLSConfigFromCertAndKey() 方法从 PEM 格式中有效的  cert/key 对中返回 tls.config 对象：


```go
// TLSConfigFromCertAndKey return a tls.config object from valid cert/key pair in PEM format.
func TLSConfigFromCertAndKey(certPem, keyPem []byte, serverName string, rootCA *x509.CertPool) (*tls.Config, error) {
	cert, err := tls.X509KeyPair(certPem, keyPem)
	if err != nil {
		return nil, err
	}

	// nolint:gosec
	config := &tls.Config{
		InsecureSkipVerify: false,
		RootCAs:            rootCA,
		ServerName:         serverName,
		Certificates:       []tls.Certificate{cert},
	}

	return config, nil
}
```
