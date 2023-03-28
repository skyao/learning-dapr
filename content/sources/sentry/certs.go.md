---
title: "certs代码"
linkTitle: "certs.go"
weight: 60
date: 2022-02-22
description: >
  处理 certs 的相关逻辑
---



certs 相关的逻辑实现在文件 `pkg/sentry/certs/certs.go` 中。



## 准备工作

### 常量定义

```go
const (
	BlockTypeCertificate     = "CERTIFICATE"
	BlockTypeECPrivateKey    = "EC PRIVATE KEY"
	BlockTypePKCS1PrivateKey = "RSA PRIVATE KEY"
	BlockTypePKCS8PrivateKey = "PRIVATE KEY"
)
```

> 备注：这里的常量定义和 csr.go 中的有部分重复。

### 结构体定义

*Credentials* 结构体包含一个证书和一个 私钥：

```go
// Credentials holds a certificate and private key.
type Credentials struct {
	PrivateKey  crypto.PrivateKey
	Certificate *x509.Certificate
}
```

## 实现

### 解码 PEM key

DecodePEMKey() 接收一个 PEM key 字节数组并返回一个代表 RSA 或 EC 私钥 的对象：

```go
func DecodePEMKey(key []byte) (crypto.PrivateKey, error) {
  // 解码 pem key
	block, _ := pem.Decode(key)
	if block == nil {
		return nil, errors.New("key is not PEM encoded")
	}
  
  // 按照类型进行后续解析处理
	switch block.Type {
	case BlockTypeECPrivateKey:
    // EC Private Key
		return x509.ParseECPrivateKey(block.Bytes)
	case BlockTypePKCS1PrivateKey:
    // PKCS1 Private Key
		return x509.ParsePKCS1PrivateKey(block.Bytes)
	case BlockTypePKCS8PrivateKey:
    // PKCS8 Private Key
		return x509.ParsePKCS8PrivateKey(block.Bytes)
	default:
		return nil, fmt.Errorf("unsupported block type %s", block.Type)
	}
}
```



### 解码 PEM 证书

DecodePEMCertificates() 方法接收一个 PEM 编码的 x509 证书字节数组，并以 x509.Certificate 对象片断的方式返回所有证书：

```go
func DecodePEMCertificates(crtb []byte) ([]*x509.Certificate, error) {
	certs := []*x509.Certificate{}
  // crtb 数组可能包含多个证书
	for len(crtb) > 0 {
		var err error
		var cert *x509.Certificate

    // 解码单个 pem 证书
		cert, crtb, err = decodeCertificatePEM(crtb)
		if err != nil {
			return nil, err
		}
		if cert != nil {
			// it's a cert, add to pool
			certs = append(certs, cert)
		}
	}
	return certs, nil
}
```

decodeCertificatePEM() 方法解码单个 pem 证书：

```go
func decodeCertificatePEM(crtb []byte) (*x509.Certificate, []byte, error) {
  // 执行pem 解码
  // pem.Decode() 方法将在输入中找到下一个 PEM 格式的块（证书，私钥  等）的输入。
  // 它返回该块和输入的其余部分。
  // 注意是返回剩余部分，当没有更多部分时，返回的长度为0
  // 如果没有找到PEM数据，则返回 block 为nil，其余部分返回整个输入。
	block, crtb := pem.Decode(crtb)
	if block == nil {
		return nil, crtb, errors.New("invalid PEM certificate")
	}
	if block.Type != BlockTypeCertificate {
		return nil, nil, nil
	}
  // 解码 x509 证书
	c, err := x509.ParseCertificate(block.Bytes)
	return c, crtb, err
}
```

生成基础证书的第一步就是生成其他证书。

### 从文件中获取 PEM 凭证

PEMCredentialsFromFiles() 方法接收一个密钥/证书对的路径，并返回一个经过验证的Credentials包装器：

```go
func PEMCredentialsFromFiles(certPem, keyPem []byte) (*Credentials, error) {
  // 解码 PEM key
	pk, err := DecodePEMKey(keyPem)
	if err != nil {
		return nil, err
	}

  // 解码 PEM 证书
  // 如果有多个证书，实际后续只使用多个证书中的第一个
	crts, err := DecodePEMCertificates(certPem)
	if err != nil {
		return nil, err
	}

	if len(crts) == 0 {
		return nil, errors.New("no certificates found")
	}

  // 检查私钥和证书的 PublicKey 是否匹配
	match := matchCertificateAndKey(pk, crts[0])
	if !match {
		return nil, errors.New("error validating credentials: public and private key pair do not match")
	}

  // 构建 Credentials 结构体并返回
	creds := &Credentials{
		PrivateKey:  pk,
		Certificate: crts[0],
	}

	return creds, nil
}
```

matchCertificateAndKey() 方法检查私钥和证书的 PublicKey 是否匹配  ：

```go
func matchCertificateAndKey(pk any, cert *x509.Certificate) bool {
  // 根据私钥的类型进行匹配
  // 实际是根据私钥类型的不同，获取到 cert 相应的 PublicKey，然后和私钥的 PublicKey 对比看是否相同
	switch key := pk.(type) {
	case *ecdsa.PrivateKey:
    // ecdsa PrivateKey
		if cert.PublicKeyAlgorithm != x509.ECDSA {
			return false
		}
		pub, ok := cert.PublicKey.(*ecdsa.PublicKey)
		return ok && pub.Equal(key.Public())
	case *rsa.PrivateKey:
    // rsa PrivateKey
		if cert.PublicKeyAlgorithm != x509.RSA {
			return false
		}
		pub, ok := cert.PublicKey.(*rsa.PublicKey)
		return ok && pub.Equal(key.Public())
	case ed25519.PrivateKey:
    // ed25519 Private Key
		if cert.PublicKeyAlgorithm != x509.Ed25519 {
			return false
		}
		pub, ok := cert.PublicKey.(ed25519.PublicKey)
		return ok && pub.Equal(key.Public())
	default:
		return false
	}
}
```

### 从 PEM 创建 cert pool

CertPoolFromPEM() 方法从一个 PEM 编码的证书字符串返回一个 CertPool 

```go
func CertPoolFromPEM(certPem []byte) (*x509.CertPool, error) {
  // 解码 PEM 证书
	certs, err := DecodePEMCertificates(certPem)
	if err != nil {
		return nil, err
	}
	if len(certs) == 0 {
		return nil, errors.New("no certificates found")
	}

  // 从多个证书中创建 cert pool
	return certPoolFromCertificates(certs), nil
}
```

certPoolFromCertificates() 方法的实现很简单：

```go
func certPoolFromCertificates(certs []*x509.Certificate) *x509.CertPool {
  // 创建 cert pool
	pool := x509.NewCertPool()
	for _, c := range certs {
    // 将每个证书添加到 pool
		pool.AddCert(c)
	}
	return pool
}
```

### 解析 PRM CSR 

ParsePemCSR() 使用给定的 PEM 编码的证书签名请求构建一个 x509 证书请求：

```go
func ParsePemCSR(csrPem []byte) (*x509.CertificateRequest, error) {
  // pem 解码
	block, _ := pem.Decode(csrPem)
	if block == nil {
		return nil, errors.New("certificate signing request is not properly encoded")
	}
  
  // 尝试 x509 解码证书请求
	csr, err := x509.ParseCertificateRequest(block.Bytes)
	if err != nil {
		return nil, fmt.Errorf("failed to parse X.509 certificate signing request: %w", err)
	}
	return csr, nil
}
```

### 生成 ECP 私钥

GenerateECPrivateKey() 方法返回一个新的 ECP 私钥：

```go
func GenerateECPrivateKey() (*ecdsa.PrivateKey, error) {
	return ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
}
```





这里涉及很多 x509 相关的领域知识。
