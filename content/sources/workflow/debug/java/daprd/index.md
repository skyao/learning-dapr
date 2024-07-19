---
title: "运行Daprd"
linkTitle: "daprd"
weight: 10
date: 2021-02-24
description: >
  以 debug 的方式运行 daprd sidecar
---

## 准备工作

在

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "daprd-workflow-OrderProcessingService",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceRoot}/cmd/daprd/main.go",
      "env": {},
      "args": [
        "--app-id=WorkflowConsoleApp", 
        "--resources-path=/home/sky/work/code/dapr-fork/quickstarts/workflows/components",
        "--dapr-http-port=32971",
        "--dapr-grpc-port=43899",
        "--app-port=",
        "--app-port=",
      ],
      "buildFlags": [
        "-tags stablecomponents"
      ]
    }
  ]
}
```

注意一定要加 buildFlags，否则会 panic，原因在 `cmd/daprd/components/zz_notag.go` 里面有强制要求，必须设置：

```go
func init() {
	panic("When building github.com/dapr/dapr/cmd/daprd, you must use either '-tags stablecomponents' or `-tags allcomponents'")
}
```


daprd 具体有哪些参数，参考 dapr/dapr 仓库下的 `cmd/daprd/options/options.go`