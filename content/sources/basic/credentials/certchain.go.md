---
type: docs
title: "certchain.go的源码学习"
linkTitle: "certchain.go"
weight: 361
date: 2021-03-12
description: >
  credentials 结构体持有证书相关的各种 path
---

Dapr credentials package中的 certchain.go 文件的源码学习，credentials 结构体持有证书相关的各种 path。



### CertChain 结构体定义

CertChain 结构体持有证书信任链的PEM值：

```go
// CertChain holds the certificate trust chain PEM values
type CertChain struct {
	RootCA []byte
	Cert   []byte
	Key    []byte
}
```

### 装载证书的LoadFromDisk 方法

LoadFromDisk 方法从给定目录中读取 CertChain：

```go
// LoadFromDisk retruns a CertChain from a given directory
func LoadFromDisk(rootCertPath, issuerCertPath, issuerKeyPath string) (*CertChain, error) {
   rootCert, err := ioutil.ReadFile(rootCertPath)
   if err != nil {
      return nil, err
   }
   cert, err := ioutil.ReadFile(issuerCertPath)
   if err != nil {
      return nil, err
   }
   key, err := ioutil.ReadFile(issuerKeyPath)
   if err != nil {
      return nil, err
   }
   return &CertChain{
      RootCA: rootCert,
      Cert:   cert,
      Key:    key,
   }, nil
}
```



### 使用场景

placement 的 main.go 中，如果 mTLS 开启了，则会读取 tls 证书：

```go
func loadCertChains(certChainPath string) *credentials.CertChain {
   tlsCreds := credentials.NewTLSCredentials(certChainPath)

   log.Info("mTLS enabled, getting tls certificates")
   // try to load certs from disk, if not yet there, start a watch on the local filesystem
   chain, err := credentials.LoadFromDisk(tlsCreds.RootCertPath(), tlsCreds.CertPath(), tlsCreds.KeyPath())
	......
}
```

operator 的 operator.go 中，也会判断，如果 MTLSEnabled :

```go
var certChain *credentials.CertChain
if o.config.MTLSEnabled {
   log.Info("mTLS enabled, getting tls certificates")
   // try to load certs from disk, if not yet there, start a watch on the local filesystem
   chain, err := credentials.LoadFromDisk(o.config.Credentials.RootCertPath(), o.config.Credentials.CertPath(), o.config.Credentials.KeyPath())
   ......
}
```

> 备注：上面两段代码重复度极高，最好能重构一下。

sentry 中也有调用：

```go
func (c *defaultCA) validateAndBuildTrustBundle() (*trustRootBundle, error) {
	var (
		issuerCreds     *certs.Credentials
		rootCertBytes   []byte
		issuerCertBytes []byte
	)

	// certs exist on disk or getting created, load them when ready
	if !shouldCreateCerts(c.config) {
		err := detectCertificates(c.config.RootCertPath)
		if err != nil {
			return nil, err
		}

		certChain, err := credentials.LoadFromDisk(c.config.RootCertPath, c.config.IssuerCertPath, c.config.IssuerKeyPath)
		if err != nil {
			return nil, errors.Wrap(err, "error loading cert chain from disk")
		}
```

> TODO: 证书相关的细节后面单独细看。