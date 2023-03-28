---
title: "spiff.go"
linkTitle: "spiff.go"
weight: 30
date: 2022-02-22
description: >
  创建 spiff ID
---



### CreateSPIFFEID 方法

CreateSPIFFEID() 方法从给定的 trustDomain, namespace, appID 创建符合 SPIFFE 标准的唯一ID：

```go
func CreateSPIFFEID(trustDomain, namespace, appID string) (string, error) {
  // trustDomain, namespace, appID 三者都不能为空
	if trustDomain == "" {
		return "", errors.New("can't create spiffe id: trust domain is empty")
	}
	if namespace == "" {
		return "", errors.New("can't create spiffe id: namespace is empty")
	}
	if appID == "" {
		return "", errors.New("can't create spiffe id: app id is empty")
	}

  // 根据 SPIFFE 规范进行验证
	// Validate according to the SPIFFE spec
	if strings.ContainsRune(trustDomain, ':') {
    // trustDomain不能带":"
		return "", errors.New("trust domain cannot contain the ':' character")
	}
  // trustDomain 的长度不能大于255个 byte
	if len([]byte(trustDomain)) > 255 {
		return "", errors.New("trust domain cannot exceed 255 bytes")
	}

  // 拼接出 SPIFFE ID
	id := fmt.Sprintf("spiffe://%s/ns/%s/%s", trustDomain, namespace, appID)
	if len([]byte(id)) > 2048 {
    // 验证 SPIFFE ID 长度不大于 2048
		return "", errors.New("spiffe id cannot exceed 2048 bytes")
	}
	return id, nil
}
```



