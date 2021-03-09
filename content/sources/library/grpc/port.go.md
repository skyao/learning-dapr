---
type: docs
title: "port.go的源码学习"
linkTitle: "port.go"
weight: 222
date: 2021-02-27
description: >
  只有一个 GetFreePort 方法用于获取一个空闲的端口。
---

Dapr grpc package中的 port.go文件的源码分析，只有一个 GetFreePort 方法用于获取一个空闲的端口。

### GetFreePort 方法

GetFreePort 方法从操作系统获取一个空闲可用的端口：

```go
// GetFreePort returns a free port from the OS
func GetFreePort() (int, error) {
	addr, err := net.ResolveTCPAddr("tcp", "localhost:0")
	if err != nil {
		return 0, err
	}

	l, err := net.ListenTCP("tcp", addr)
	if err != nil {
		return 0, err
	}
	defer l.Close()
	return l.Addr().(*net.TCPAddr).Port, nil
}
```

通过将端口设置为0, 来让操作系统自动分配一个可用的端口。注意返回时一定要关闭这个连接。
