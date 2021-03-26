---
type: docs
title: "credentials.go的源码学习"
linkTitle: "credentials.go"
weight: 361
date: 2021-03-12
description: >
  credentials 结构体持有证书相关的各种 path
---

Dapr credentials package中的 credentials.go文件的源码学习，credentials 结构体持有证书相关的各种 path。

### TLSCredentials 结构体定义

只有一个字段 credentialsPath：

```go
// TLSCredentials holds paths for credentials
type TLSCredentials struct {
   credentialsPath string
}
```

构造方法很简单：

```go
// NewTLSCredentials returns a new TLSCredentials
func NewTLSCredentials(path string) TLSCredentials {
   return TLSCredentials{
      credentialsPath: path,
   }
}
```

### 获取相关 path 的方法

获取 credentialsPath，这个path中保存有 TLS 证书：

```go
// Path returns the directory holding the TLS credentials
func (t *TLSCredentials) Path() string {
   return t.credentialsPath
}
```

分别获取 root cert / cert / cert key 的 path：

```go
// RootCertPath returns the file path for the root cert
func (t *TLSCredentials) RootCertPath() string {
   return filepath.Join(t.credentialsPath, RootCertFilename)
}

// CertPath returns the file path for the cert
func (t *TLSCredentials) CertPath() string {
   return filepath.Join(t.credentialsPath, IssuerCertFilename)
}

// KeyPath returns the file path for the cert key
func (t *TLSCredentials) KeyPath() string {
   return filepath.Join(t.credentialsPath, IssuerKeyFilename)
}
```