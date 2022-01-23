---
type: docs
title: "util.go的源码学习"
linkTitle: "util.go"
weight: 221
date: 2021-02-27
description: >
  目前只有用于转换state参数类型的两个方法
---

Dapr grpc package中的 util.go文件的源码分析，目前只有用于转换state参数类型的两个方法。

### stateConsistencyToString 方法

stateConsistencyToString 方法将 StateOptions_StateConsistency 转为 string：

```go
func stateConsistencyToString(c commonv1pb.StateOptions_StateConsistency) string {
	switch c {
	case commonv1pb.StateOptions_CONSISTENCY_EVENTUAL:
		return "eventual"
	case commonv1pb.StateOptions_CONSISTENCY_STRONG:
		return "strong"
	}

	return ""
}
```

### stateConcurrencyToString 方法

方法 方法将 StateOptions_StateConsistency 转为 string：

```go
func stateConcurrencyToString(c commonv1pb.StateOptions_StateConcurrency) string {
	switch c {
	case commonv1pb.StateOptions_CONCURRENCY_FIRST_WRITE:
		return "first-write"
	case commonv1pb.StateOptions_CONCURRENCY_LAST_WRITE:
		return "last-write"
	}

	return ""
}
```

