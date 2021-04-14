---
type: docs
title: "channel.go的源码学习"
linkTitle: "channel.go"
weight: 1810
date: 2021-04-11
description: >
  定义 AppChannel 接口和方法
---

Dapr channel package中的 channel.go 文件的源码学习，定义 AppChannel 接口和方法。

**AppChannel 是和用户代码进行通讯的抽象。**

常量定义 DefaultChannelAddress，考虑到 dapr 通常是以 sidecar 模式部署的，因此默认channel 地址是 127.0.0.1

```go
const (
   // DefaultChannelAddress is the address that user application listen to
   DefaultChannelAddress = "127.0.0.1"
)
```

方法定义：

```go
// AppChannel is an abstraction over communications with user code
type AppChannel interface {
   GetBaseAddress() string
   InvokeMethod(ctx context.Context, req *invokev1.InvokeMethodRequest) (*invokev1.InvokeMethodResponse, error)
}
```