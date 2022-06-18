---
title: "kafka组件配置"
linkTitle: "kafka组件配置"
weight: 30
date: 2022-04-26
description: >
  kafka pubsub component认证测试中的kafka组件配置
---



## 组件配置文件

kafka pubsub 组件认证测试中使用到三个kafka pubsub 组件：

1. consumer1
2. mtls-consumer
3. oauth-consumer



![kafka-component-files](images/kafka-component-files.png)



### 共性配置

三个 component 的 name 都设定为 messagebus：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
spec:
  type: pubsub.kafka
  version: v1
  ......
  metadata:
  - name: initialOffset
    value: oldest
  - name: backOffDuration
    value: 50ms
```

### consumer1

```yaml
  metadata:
  - name: brokers
    value: localhost:19092,localhost:29092,localhost:39092
  - name: consumerGroup
    value: kafkaCertification1
  - name: authType
    value: "none"
```

- brokers 指向端口为 19092 / 29092 / 39092 的kafka集群

- authType 设置为 none，不进行authentication。
- consumerGroup 设置为 kafkaCertification1

### mtls-consumer

```yaml
  metadata:
  - name: brokers
    value: localhost:19094,localhost:29094,localhost:39094
  - name: consumerGroup
    value: kafkaCertification2
  - name: authType
    value: mtls
  - name: caCert
    value: |
      -----BEGIN CERTIFICATE-----
			......
      -----END CERTIFICATE-----
  - name: clientCert
    value: |
      -----BEGIN CERTIFICATE-----
			......
      -----END CERTIFICATE-----
  - name: clientKey
    value: |
      -----BEGIN RSA PRIVATE KEY-----
      ......
      -----END RSA PRIVATE KEY-----
```

- brokers 指向端口为 19094 / 29094 / 39094 的kafka集群

- authType 设置为 mtls，进行mtls authentication。
- consumerGroup 设置为 kafkaCertification2

### oauth-consumer

```yaml
  metadata:
  - name: brokers
    value: localhost:19093,localhost:29093,localhost:39093
  - name: consumerGroup
    value: kafkaCertification2
  - name: authType
    value: "oidc"
  - name: oidcTokenEndpoint
    value: https://localhost:4443/oauth2/token
  - name: oidcClientID
    value: "dapr"
  - name: oidcClientSecret
    value: "dapr-test"
  - name: oidcScopes
    value: openid,kafka
  - name: caCert
    value: |
      -----BEGIN CERTIFICATE-----
			......
      -----END CERTIFICATE-----
```

- brokers 指向端口为 19093 / 29093 / 39093 的kafka集群

- authType 设置为 oidc，进行auth authentication。
- consumerGroup 设置为 kafkaCertification2 (和 mtls 一致？)
