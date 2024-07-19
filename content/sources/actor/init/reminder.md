---
title: "reminder初始化过程"
linkTitle: "reminder"
weight: 30
date: 2021-02-24
description: >
  reminder 初始化
---

## reminder init()

reminder 初始化代码在 `pkg/actors/reminders/reminders.go`，但是内容为空。

```go
func (r *reminders) Init(ctx context.Context) error {
	return nil
}
```