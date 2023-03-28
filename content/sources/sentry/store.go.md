---
title: "certs store代码"
linkTitle: "store.go"
weight: 60
date: 2022-02-22
description: >
  实现 certs 的存储 store
---



certs 相关的存储逻辑实现在文件 `pkg/sentry/certs/store.go` 中。



## 准备工作

### 常量定义

```go
const (
	defaultSecretNamespace = "default"
)
```



## 实现

### 存储凭证

StoreCredentials() 方法将 trust bundle 存储在 Kubernetes secret store  或者本地磁盘上，取决于托管的平台：

```go
func StoreCredentials(ctx context.Context, conf config.SentryConfig, rootCertPem, issuerCertPem, issuerKeyPem []byte) error {
	if config.IsKubernetesHosted() {
    // 如果是 k8s 托管来
		return storeKubernetes(ctx, rootCertPem, issuerCertPem, issuerKeyPem)
	}
  
  // 否则是自托管
	return storeSelfhosted(rootCertPem, issuerCertPem, issuerKeyPem, conf.RootCertPath, conf.IssuerCertPath, conf.IssuerKeyPath)
}
```

#### 在 Kubernetes 中的存储

storeKubernetes() 方法将凭证存储在 Kubernetes secret store 中：

```go
// 部分常量定于在 consts.go 中
const (
	TrustBundleK8sSecretName = "dapr-trust-bundle" /* #nosec */
)

func storeKubernetes(ctx context.Context, rootCertPem, issuerCertPem, issuerCertKey []byte) error {
  // 准备 k8s client
	kubeClient, err := kubernetes.GetClient()
	if err != nil {
		return err
	}

  // 获取 namespace
	namespace := getNamespace()
  // 调用 k8s API 的方法获取 secret
	secret, err := kubeClient.CoreV1().Secrets(namespace).Get(context.TODO(), consts.TrustBundleK8sSecretName, metav1.GetOptions{})
	if errors.IsNotFound(err) {
		return fmt.Errorf("failed to get secret %w", err)
	}

  // 将 rootCertPem / issuerCertPem / issuerCertKey 保存到 secret 的 Data 中
	secret.Data = map[string][]byte{
		credentials.RootCertFilename:   rootCertPem,
		credentials.IssuerCertFilename: issuerCertPem,
		credentials.IssuerKeyFilename:  issuerCertKey,
	}

  // 更新保存 secret
	// We update and not create because sentry expects a secret to already exist
	_, err = kubeClient.CoreV1().Secrets(namespace).Update(ctx, secret, metav1.UpdateOptions{})
	if err != nil {
		return fmt.Errorf("failed saving secret to kubernetes: %w", err)
	}
	return nil
}
```

其中 getNamespace() 读取环境变量 "NAMESPACE" 来获知当前的命名空间，缺省值为 "default"：

```bash
const (
	defaultSecretNamespace = "default"
)

func getNamespace() string {
	namespace := os.Getenv("NAMESPACE")
	if namespace == "" {
		namespace = defaultSecretNamespace
	}
	return namespace
}
```

#### 自托管时的存储

storeSelfhosted() 方法将凭证存储在本地文件中：

```go
func StoreCredentials(...) {
  ......
	return storeSelfhosted(rootCertPem, issuerCertPem, issuerKeyPem, conf.RootCertPath, conf.IssuerCertPath, conf.IssuerKeyPath)
  }

func storeSelfhosted(rootCertPem, issuerCertPem, issuerKeyPem []byte, rootCertPath, issuerCertPath, issuerKeyPath string) error {
  // 分别将三个内容保存到三个文件中
	err := os.WriteFile(rootCertPath, rootCertPem, 0o644)
	if err != nil {
		return fmt.Errorf("failed saving file to %s: %w", rootCertPath, err)
	}

	err = os.WriteFile(issuerCertPath, issuerCertPem, 0o644)
	if err != nil {
		return fmt.Errorf("failed saving file to %s: %w", issuerCertPath, err)
	}

	err = os.WriteFile(issuerKeyPath, issuerKeyPem, 0o644)
	if err != nil {
		return fmt.Errorf("failed saving file to %s: %w", issuerKeyPath, err)
	}
	return nil
}
```

rootCertPem / issuerCertPem / issuerKeyPem 分别保存到 conf.RootCertPath / conf.IssuerCertPath / conf.IssuerKeyPath 这三个 sentry 配置指定的文件路径中。

回顾一下 main.go  中读取相关配置的代码实现：

```go
const (
	defaultCredentialsPath = "/var/run/dapr/credentials"
)

var (
	// RootCertFilename is the filename that holds the root certificate.
	RootCertFilename = "ca.crt"
	// IssuerCertFilename is the filename that holds the issuer certificate.
	IssuerCertFilename = "issuer.crt"
	// IssuerKeyFilename is the filename that holds the issuer key.
	IssuerKeyFilename = "issuer.key"
)

func main() {
  ......
credsPath := flag.String("issuer-credentials", defaultCredentialsPath, "Path to the credentials directory holding the issuer data")	
  flag.StringVar(&credentials.RootCertFilename, "issuer-ca-filename", credentials.RootCertFilename, "Certificate Authority certificate filename")
	flag.StringVar(&credentials.IssuerCertFilename, "issuer-certificate-filename", credentials.IssuerCertFilename, "Issuer certificate filename")
	flag.StringVar(&credentials.IssuerKeyFilename, "issuer-key-filename", credentials.IssuerKeyFilename, "Issuer private key filename")

	issuerCertPath := filepath.Join(*credsPath, credentials.IssuerCertFilename)
	issuerKeyPath := filepath.Join(*credsPath, credentials.IssuerKeyFilename)
	rootCertPath := filepath.Join(*credsPath, credentials.RootCertFilename)

	......
	config.IssuerCertPath = issuerCertPath
	config.IssuerKeyPath = issuerKeyPath
	config.RootCertPath = rootCertPath
  ......
}
```

可见默认是使用 "/var/run/dapr/credentials" 目录下的这三个文件：

- "ca.crt"
- "issuer.crt"
- "issuer.key"

