---
title: "timer初始化过程"
linkTitle: "timer"
weight: 40
date: 2021-02-24
description: >
  timer 初始化
---

## timer init()

reminder 初始化代码在 `pkg/actors/timers/timers.go`，但是内容为空。

```go
func (t *timers) Init(ctx context.Context) error {
	return nil
}
```