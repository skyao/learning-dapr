---
type: docs
title: "dial.go的源码学习"
linkTitle: "dial.go"
weight: 223
date: 2021-03-09
description: >
  目前只有用于建连获取地址前缀的一个方法
---

Dapr grpc package中的 dial.go文件的源码分析，目前只有用于建连获取地址前缀的一个方法。

### GetDialAddressPrefix 方法

GetDialAddressPrefix 为给定的 DaprMode 返回 dial 前缀，用于gPRC 客户端连接：

```go
// GetDialAddressPrefix returns a dial prefix for a gRPC client connections
// For a given DaprMode.
func GetDialAddressPrefix(mode modes.DaprMode) string {
	if runtime.GOOS == "windows" {
		return ""
	}

	switch mode {
	case modes.KubernetesMode:
		return "dns:///"
	default:
		return ""
	}
}
```

注意：Kubernetes 模式下 返回 "dns:///" 

调用场景，只在 grpc.go 的 GetGRPCConnection() 方法中被调用：

```go
// GetGRPCConnection returns a new grpc connection for a given address and inits one if doesn't exist
func (g *Manager) GetGRPCConnection(address, id string, namespace string, skipTLS, recreateIfExists, sslEnabled bool) (*grpc.ClientConn, error) {
    dialPrefix := GetDialAddressPrefix(g.mode)
    ......
    conn, err := grpc.DialContext(ctx, dialPrefix+address, opts...)
    ......
}
```

