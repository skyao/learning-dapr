---
title: "为lint做准备"
linkTitle: "lint"
weight: 40
date: 2022-04-08
description: >
  安装配置 golangci-lint、gofumpt、goimports
---

## golangci-lint 准备

### golangci-lint 的背景

dapr提供了 `make lint`  target 来执行  golangci-lint， 如果没有安装 golangci-lint 则会报错：

```bash
$ cd ~/work/code/dapr/dapr
$ make lint       
......
Error getting current version: exec: "golangci-lint": executable file not found in $PATH
Please install the correct version using the guide - https://golangci-lint.run/usage/install/
exit status 1
make: *** [Makefile:390: check-linter] Error 1
```

注意： 必须和 dapr 使用的 golangci-lint 版本保持一致。

### 如何知道 Dapr 使用的 golangci-lint 的版本？

查看 dapr 仓库下的 Makefile 文件： 

https://github.com/dapr/dapr/blob/master/Makefile

找到下述内容：

```bash
# Please use golangci-lint version v1.55.2 , otherwise you might encounter errors.
```

查看 .github/workflows/dapr.yml 文件：

https://github.com/dapr/dapr/blob/master/.github/workflows/dapr.yml

找到下述内容：

```yaml
env:
      GOLANGCILINT_VER: "v1.55.2"
```

就可以得到目前 dapr 使用的版本是 "v1.55.2"。

## 安装 golangci-lint

安装方式参考 https://golangci-lint.run/usage/install/ 。

在release页面找到对应版本，主要不要直接安装最新版本：

https://github.com/golangci/golangci-lint/releases/tag/v1.55.2

### linux 安装

linux下执行如下命令：

```bash
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.55.2 # 特别注意这里一定要指定正确的版本
golangci/golangci-lint info checking GitHub for tag 'v1.55.2'
golangci/golangci-lint info found version: 1.55.2 for v1.55.2/linux/amd64
golangci/golangci-lint info installed /home/sky/work/soft/gopath/bin/golangci-lint


$ golangci-lint --version
golangci-lint has version 1.55.2 built with go1.21.3 from e3c2265f on 2023-11-03T12:59:25Z
```

{{% alert title="切记" color="warning" %}}
golangci-lint 一定要安装对应的版本！
{{% /alert %}}

### m1 macbook上安装

在 m1 macbook 上， dapr之前使用的 1.31 版本发布较早，没有提供对 m1 （也就是darwin-arm64）的支持，但最新的dapr改用 1.45.2 版本之后就支持 arm64 了，所以可以用同样的方式安装：

```bash
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.52.1
golangci/golangci-lint info checking GitHub for tag 'v1.52.1'
golangci/golangci-lint info found version: 1.52.1 for v1.52.1/linux/amd64
golangci/golangci-lint info installed /home/sky/work/soft/gopath/bin/golangci-lint
 
$ golangci-lint --version
golangci-lint has version 1.52.1 built with go1.20.2 from d92b38cc on 2023-03-21T19:48:38Z
```

注意：golangci-lint  新版本似对 go 有版本要求，如果遇到报错，如我在 golang 1.17 版本下运行会报错：

```bash
$ golangci-lint --version

panic: load embedded ruleguard rules: rules/rules.go:13: can't load fmt

goroutine 1 [running]:
github.com/go-critic/go-critic/checkers.init.22()
	github.com/go-critic/go-critic@v0.6.2/checkers/embedded_rules.go:46 +0x494

$ go version  
go version go1.17.8 darwin/arm64
```

升级 golang 到新版本如 1.18  就正常了。

## gofumpt

运行 lint 之后如果发现 `File is not `gofumpt`-ed` ：

```bash
$ make lint
golangci-lint run --timeout=20m

tests/perf/utils/grpc_helpers.go:4               gofumpt    File is not `gofumpt`-ed
tests/perf/utils/grpc_helpers.go:9               gofumpt    File is not `gofumpt`-ed
```

则需要安装 gofumpt 进行文件格式，参考 https://github.com/mvdan/gofumpt ：

```bash
$ go install mvdan.cc/gofumpt@latest
go: downloading golang.org/x/sync v0.0.0-20220819030929-7fc1605a5dde
go: downloading golang.org/x/sys v0.0.0-20220829200755-d48e67d00261
```

然后对有问题的文件执行 gofumpt :

```bash
gofumpt -w tests/perf/utils/grpc_helpers.go 
```


## goimports

运行 lint 之后如果发现 `File is not `goimports`-ed` ：

```bash
$ make lint
golangci-lint run --timeout=20m

tests/perf/utils/grpc_helpers.go:5               goimports  File is not `goimports`-ed with -local github.com/dapr/
```

则需要安装 goimports 对import内容进行文件格式：

```bash
go get golang.org/x/tools/cmd/goimports
```

### 手工执行

```bash
$ goimports -e -d -local github.com/dapr/ tests/perf/utils/grpc_helpers.go
diff -u tests/perf/utils/grpc_helpers.go.orig 
tests/perf/utils/grpc_helpers.go
--- tests/perf/utils/grpc_helpers.go.orig       2022-04-08 22:47:08.199473748 +0800
+++ tests/perf/utils/grpc_helpers.go    2022-04-08 22:47:08.199473748 +0800
@@ -4,10 +4,11 @@
        "context"
        "time"
 
-       v1 "github.com/dapr/dapr/pkg/proto/common/v1"
-       runtimev1pb "github.com/dapr/dapr/pkg/proto/runtime/v1"
        "google.golang.org/grpc"
        "google.golang.org/protobuf/types/known/anypb"
+
+       v1 "github.com/dapr/dapr/pkg/proto/common/v1"
+       runtimev1pb "github.com/dapr/dapr/pkg/proto/runtime/v1"
 )
 
 $ goimports -w tests/perf/utils/grpc_helpers.go
```

### 配置vs code

参考： [Formatting Go code with goimports (hyr.mn)](https://hyr.mn/gofmt/)

打开 File -> Preferences -> Settings， 搜索 "format on save"，勾选:

![vscode-format-on-save](images/vscode-format-on-save.png)

搜索 go:format，在 format tool 中选择 goimports: 

![vscode-goimports](images/vscode-goimports.png)

注意：

1. 如果 go:format 搜索时找不到 extends / go，则应该是没有安装 go extensin，或者安装之后没有重启 vs code
2. 下拉框选 goimports 时如果报错没有安装 goimports，点安装即可

### 参考资料

- [How to Fix some golangci-lint errors](http://giaogiaocat.github.io/go/how-to-fix-file-is-not-gofumpt-ed-gofumpt-error/)
