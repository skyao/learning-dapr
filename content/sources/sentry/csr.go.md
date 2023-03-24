---
title: "csr代码"
linkTitle: "csr.go"
weight: 50
date: 2022-02-22
description: >
  处理 csr 的相关逻辑
---



csr 相关的逻辑实现在文件 `pkg/sentry/csr/csr.go` 中。



## 准备工作

### 常量定义

```go
const (
	blockTypeECPrivateKey = "EC PRIVATE KEY" // EC private key
	blockTypePrivateKey   = "PRIVATE KEY"    // PKCS#8 private key
	encodeMsgCSR          = "CERTIFICATE REQUEST"
	encodeMsgCert         = "CERTIFICATE"
)
```

### 全局变量定义

```go
// The OID for the SAN extension (http://www.alvestrand.no/objectid/2.5.29.17.html)
var oidSubjectAlternativeName = asn1.ObjectIdentifier{2, 5, 29, 17}
```

## 实现

### 生成CSR

GenerateCSR() f方法创建 X.509 certificate sign request 和私钥：

```go
// GenerateCSR creates a X.509 certificate sign request and private key.
func GenerateCSR(org string, pkcs8 bool) ([]byte, []byte, error) {
  // 生成 ec 私钥
	key, err := certs.GenerateECPrivateKey()
	if err != nil {
		return nil, nil, fmt.Errorf("unable to generate private keys: %w", err)
	}

  // 生成 csr 模版
	templ, err := genCSRTemplate(org)
	if err != nil {
		return nil, nil, fmt.Errorf("error generating csr template: %w", err)
	}

  // 创建证书请求
	csrBytes, err := x509.CreateCertificateRequest(rand.Reader, templ, key)
	if err != nil {
		return nil, nil, fmt.Errorf("failed to create CSR: %w", err)
	}

  // 编码证书
	crtPem, keyPem, err := encode(true, csrBytes, key, pkcs8)
	return crtPem, keyPem, err
}
```

生成 csr 模版的实现，只设置了Organization ：

```go
func genCSRTemplate(org string) (*x509.CertificateRequest, error) {
	return &x509.CertificateRequest{
		Subject: pkix.Name{
			Organization: []string{org},
		},
	}, nil
}
```

编码证书的实现代码：

```go
func encode(csr bool, csrOrCert []byte, privKey *ecdsa.PrivateKey, pkcs8 bool) ([]byte, []byte, error) {
	// 判断是 "CERTIFICATE" 还是 "CERTIFICATE REQUEST"
  encodeMsg := encodeMsgCert
	if csr {
		encodeMsg = encodeMsgCSR
	}
  // 执行编码
	csrOrCertPem := pem.EncodeToMemory(&pem.Block{Type: encodeMsg, Bytes: csrOrCert})

	var encodedKey, privPem []byte
	var err error

	if pkcs8 {
    // 如果是 pkcs8，需要将私钥编码为 PKCS8 私钥 / "PRIVATE KEY"
		if encodedKey, err = x509.MarshalPKCS8PrivateKey(privKey); err != nil {
			return nil, nil, err
		}
    // 将上面的 PKCS8 私钥编码到内存
		privPem = pem.EncodeToMemory(&pem.Block{Type: blockTypePrivateKey, Bytes: encodedKey})
	} else {
    // 不是 pkcs8 的话，需要将私钥编码为 EC 私钥 / "EC PRIVATE KEY"
		encodedKey, err = x509.MarshalECPrivateKey(privKey)
		if err != nil {
			return nil, nil, err
		}
		privPem = pem.EncodeToMemory(&pem.Block{Type: blockTypeECPrivateKey, Bytes: encodedKey})
	}
	return csrOrCertPem, privPem, nil
}
```



### 生成基础证书

generateBaseCert() 方法返回一个基本的非CA证书，该证书可以通过添加 subject、key usage 和附加属性成为一个工作负载或CA证书：

```go
// generateBaseCert returns a base non-CA cert that can be made a workload or CA cert
// By adding subjects, key usage and additional proerties.
func generateBaseCert(ttl, skew time.Duration, publicKey interface{}) (*x509.Certificate, error) {
  // 创建一个新的序列号
	serNum, err := newSerialNumber()
	if err != nil {
		return nil, err
	}

	now := time.Now().UTC()
	// Allow for clock skew with the NotBefore validity bound.
  // 允许在 NotBefore 有效期内出现时钟偏移。
	notBefore := now.Add(-1 * skew)
	notAfter := now.Add(ttl)

  // 创建并返回 x509 证书
	return &x509.Certificate{
		SerialNumber: serNum,
		NotBefore:    notBefore,
		NotAfter:     notAfter,
		PublicKey:    publicKey,
	}, nil
}
```

创建一个新的序列号的代码实现细节：

```go
func newSerialNumber() (*big.Int, error) {
  // 序列号的最大值，1 << 128
	serialNumLimit := new(big.Int).Lsh(big.NewInt(1), 128)
  // 在这个区间内取随机数
	serialNum, err := rand.Int(rand.Reader, serialNumLimit)
	if err != nil {
		return nil, fmt.Errorf("error generating serial number: %w", err)
	}
	return serialNum, nil
}
```

生成基础证书的第一步就是生成其他证书。

### 生成 Root Cert CSR

GenerateRootCertCSR() 方法返回 CA root cert x509 证书：

```go
// GenerateRootCertCSR returns a CA root cert x509 Certificate.
func GenerateRootCertCSR(org, cn string, publicKey interface{}, ttl, skew time.Duration) (*x509.Certificate, error) {
  // 先生成基本证书
	cert, err := generateBaseCert(ttl, skew, publicKey)
	if err != nil {
		return nil, err
	}

  // 设置证书的参数
	cert.KeyUsage = x509.KeyUsageCertSign
	cert.ExtKeyUsage = append(cert.ExtKeyUsage, x509.ExtKeyUsageServerAuth, x509.ExtKeyUsageClientAuth)
	cert.Subject = pkix.Name{
		CommonName:   cn,
		Organization: []string{org},
	}
	cert.DNSNames = []string{cn}
	cert.IsCA = true
	cert.BasicConstraintsValid = true
	cert.SignatureAlgorithm = x509.ECDSAWithSHA256
	return cert, nil
}
```

### 生成 CSR Certificate

GenerateCSRCertificate() 方法 返回 x509 Certificate，输入为  CSR / 签名证书 / 公钥 / 签名私钥 和持续时间：

```go
// GenerateCSRCertificate returns an x509 Certificate from a CSR, signing cert, public key, signing private key and duration.
func GenerateCSRCertificate(csr *x509.CertificateRequest, subject string, identityBundle *identity.Bundle, signingCert *x509.Certificate, publicKey interface{}, signingKey crypto.PrivateKey,
	ttl, skew time.Duration, isCA bool,
) ([]byte, error) {
  // 先生成基本证书
	cert, err := generateBaseCert(ttl, skew, publicKey)
	if err != nil {
		return nil, fmt.Errorf("error generating csr certificate: %w", err)
	}
	if isCA {
		cert.KeyUsage = x509.KeyUsageCertSign | x509.KeyUsageCRLSign
	} else {
		cert.KeyUsage = x509.KeyUsageDigitalSignature | x509.KeyUsageKeyEncipherment
		cert.ExtKeyUsage = append(cert.ExtKeyUsage, x509.ExtKeyUsageServerAuth, x509.ExtKeyUsageClientAuth)
	}

	if subject == "cluster.local" {
		cert.Subject = pkix.Name{
			CommonName: subject,
		}
		cert.DNSNames = []string{subject}
	}

	cert.Issuer = signingCert.Issuer
	cert.IsCA = isCA
	cert.IPAddresses = csr.IPAddresses
	cert.Extensions = csr.Extensions
	cert.BasicConstraintsValid = true
	cert.SignatureAlgorithm = csr.SignatureAlgorithm

	if identityBundle != nil {
		spiffeID, err := identity.CreateSPIFFEID(identityBundle.TrustDomain, identityBundle.Namespace, identityBundle.ID)
		if err != nil {
			return nil, fmt.Errorf("error generating spiffe id: %w", err)
		}

		rv := []asn1.RawValue{
			{
				Bytes: []byte(spiffeID),
				Class: asn1.ClassContextSpecific,
				Tag:   asn1.TagOID,
			},
			{
				Bytes: []byte(fmt.Sprintf("%s.%s.svc.cluster.local", subject, identityBundle.Namespace)),
				Class: asn1.ClassContextSpecific,
				Tag:   2,
			},
		}

		b, err := asn1.Marshal(rv)
		if err != nil {
			return nil, fmt.Errorf("failed to marshal asn1 raw value for spiffe id: %w", err)
		}

		cert.ExtraExtensions = append(cert.ExtraExtensions, pkix.Extension{
			Id:       oidSubjectAlternativeName,
			Value:    b,
			Critical: true, // According to x509 and SPIFFE specs, a SubjAltName extension must be critical if subject name and DNS are not present.
		})
	}

	return x509.CreateCertificate(rand.Reader, cert, signingCert, publicKey, signingKey)
}
```



这里涉及很多 x509 相关的领域知识。
