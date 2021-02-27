---
type: docs
title: "dapr_logger.go的源码分析"
linkTitle: "dapr_logger.go"
weight: 112
date: 2021-02-27
description: >
  Dapr logger package中的dapr_logger.go文件的源码分析
---



### daprLogger 结构体

daprLogger 结构体实现 

```go
// daprLogger is the implemention for logrus
type daprLogger struct {
	// name is the name of logger that is published to log as a scope
	name string
	// loger is the instance of logrus logger
	logger *logrus.Entry
}
```

