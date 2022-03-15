---
title: "Makefile文件"
linkTitle: "Makefile"
weight: 10
date: 2022-01-21
description: >
  定义和获取变量，定义 test/lint/go.mod/check-diff等target
---

https://github.com/dapr/kit/blob/main/Makefile

## 变量定义

先是一堆变量定义。

### golang变量

```makefile
export GO111MODULE ?= on
export GOPROXY ?= https://proxy.golang.org
export GOSUMDB ?= sum.golang.org
# By default, disable CGO_ENABLED. See the details on https://golang.org/cmd/cgo
CGO         ?= 0

# GOARCH 和 GOOS 的定义位置在后面一些
export GOARCH ?= $(TARGET_ARCH_LOCAL)
export GOOS ?= $(TARGET_OS_LOCAL)
```

"?=" 是 Makefile 中的赋值符号，语义是：如果之前没有赋值就赋予等号后面的值，如果之前有赋值就忽略。等价于：

```go
if (GO111MODULE == null || GO111MODULE.length() == 0) {
    GO111MODULE = "on"
}
```

参考： [Makefile 中:= ?= += =的区别](https://www.cnblogs.com/wanqieddy/archive/2011/09/21/2184257.html)

`GOPROXY` 容易理解，为了加速我们一般设置为国内的代理，典型如 `goproxy.cn`。构建中如果发现下载go package的速度不理想，可以通过设置 goproxy 来优化。

`GO111MODULE` 如果没有特别设置，则 dapr 会要求设置为 on。关于 `GO111MODULE` 这个设置的详细介绍，请参考：

- [Go 模块解惑：到处都是 GO111MODULE ，这到底什么？](https://learnku.com/go/t/39086)
- [go语言：环境变量GOPROXY和GO111MODULE设置](https://segmentfault.com/a/1190000040726392)

`GOSUMDB` 是为了保证开发者的依赖库不被人恶意劫持篡改，Go 会自动去这个服务器进行数据校验，这里取默认值就好，或者为了优化在国内的速度，可以设置为 `sum.golang.google.cn`。参考：

- [GOSUMDB 环境变量](https://goproxy.io/zh/docs/GOSUMDB-env.html)

`CGO` 默认是关闭的。

### git变量

定义了两个git相关的变量：

```makefile
GIT_COMMIT  = $(shell git rev-list -1 HEAD)
GIT_VERSION = $(shell git describe --always --abbrev=7 --dirty)
```

这两行代码相当于执行了下面两条git命令，然后将结果赋值给了 GIT_COMMIT 和 GIT_VERSION 变量：

```bash
$ git rev-list -1 HEAD
29c3d7dee532e510de7adc31bc43a21535b5e603

$ git describe --always --abbrev=7 --dirty
29c3d7d
```

对比 `git log` 命令的输入：

```bash
$ git log

commit 29c3d7dee532e510de7adc31bc43a21535b5e603 (HEAD -> main, origin/main, origin/HEAD)
Author: Will <witsai@microsoft.com>
Date:   Thu Dec 30 07:09:28 2021 -0800
......
commit 867d7d9f3e6454864b4357941bab7601ae1cbd0a
Author: Dmitry Shmulevich <dmitry.shmulevich@gmail.com>
Date:   Thu Dec 9 12:03:02 2021 -0800
......
commit db8c68dfecc9515c6da148e9439730719f838d67
Author: greenie-msft <56556602+greenie-msft@users.noreply.github.com>
Date:   Wed Nov 24 10:40:17 2021 -080
......
```

GIT_COMMIT 和 GIT_VERSION 拿到的是最新的一次 git commit  记录，只是 GIT_COMMIT 是完整的 commit，而 GIT_VERSION 是 commit 的前8个字符。

### 操作系统变量

通过 `uname -m` 命令来 LOCAL_ARCH 变量，然后判断一下，设置 TARGET_ARCH_LOCAL 变量：

```makefile
LOCAL_ARCH := $(shell uname -m)
ifeq ($(LOCAL_ARCH),x86_64)
	TARGET_ARCH_LOCAL=amd64
else ifeq ($(shell echo $(LOCAL_ARCH) | head -c 5),armv8)
	TARGET_ARCH_LOCAL=arm64
else ifeq ($(shell echo $(LOCAL_ARCH) | head -c 4),armv)
	TARGET_ARCH_LOCAL=arm
else
	TARGET_ARCH_LOCAL=amd64
endif
```

然后是类似的获取 TARGET_OS_LOCAL 变量：

```makefile
LOCAL_OS := $(shell uname)
ifeq ($(LOCAL_OS),Linux)
   TARGET_OS_LOCAL = linux
else ifeq ($(LOCAL_OS),Darwin)
   TARGET_OS_LOCAL = darwin
else
   TARGET_OS_LOCAL ?= windows
endif
```

在得到 TARGET_ARCH_LOCAL 和 TARGET_OS_LOCAL 之后，就可以设置 GOARCH 和  GOOS 了：

```makefile
export GOARCH ?= $(TARGET_ARCH_LOCAL)
export GOOS ?= $(TARGET_OS_LOCAL)
```

一些额外设置，对于windows系统会有所不同:

```makefile
ifeq ($(GOOS),windows)
BINARY_EXT_LOCAL:=.exe
GOLANGCI_LINT:=golangci-lint.exe
# Workaround for https://github.com/golang/go/issues/40795
BUILDMODE:=-buildmode=exe
else
BINARY_EXT_LOCAL:=
GOLANGCI_LINT:=golangci-lint
endif
```

## build target

然后就是正式的 build target 定义了。

### Target: test 

```makefile
################################################################################
# Target: test                                                                 #
################################################################################
.PHONY: test
test:
	go test ./... $(COVERAGE_OPTS) $(BUILDMODE)
```

test target 执行测试，实际调用 go test，执行结果如下：

```bash
$ make test 
go test ./...  
ok      github.com/dapr/kit/config      0.008s
ok      github.com/dapr/kit/logger      0.007s
ok      github.com/dapr/kit/retry       0.006s
$ make test
go test ./...  
ok      github.com/dapr/kit/config      (cached)
ok      github.com/dapr/kit/logger      (cached)
ok      github.com/dapr/kit/retry       (cached)
```

注意第二次时会有cache，如果代码和测试代码都没有修改则不会真正的跑测试，而是给出上次缓存的结果。如果要强制重新跑全部的测试，可以先做一下clean：

```bash
go clean --testcache
```

### Target: lint 

```makefile
################################################################################
# Target: lint                                                                 #
################################################################################
.PHONY: lint
lint:
	# Due to https://github.com/golangci/golangci-lint/issues/580, we need to add --fix for windows
	$(GOLANGCI_LINT) run --timeout=20m
```

在本地执行 lint target之前，先要确认已经安装了 golangci-lint ，如果没有安装则会报错：

```bash
 make lint
# Due to https://github.com/golangci/golangci-lint/issues/580, we need to add --fix for windows
golangci-lint run --timeout=20m
make: golangci-lint: Command not found
make: *** [Makefile:72: lint] Error 127
```

安装方式参考 https://golangci-lint.run/usage/install/ 。linux下执行如下命令：

```bash
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.43.0
golangci/golangci-lint info checking GitHub for tag 'v1.43.0'
golangci/golangci-lint info found version: 1.43.0 for v1.43.0/linux/amd64
golangci/golangci-lint info installed /home/sky/work/soft/gopath/bin/golangci-lint

$ golangci-lint --version
golangci-lint has version 1.43.0 built from 861262b7 on 2021-11-03T11:57:46Z
```

再次执行 make lint 进行检查，发现一堆输出：

```bash
make lint
# Due to https://github.com/golangci/golangci-lint/issues/580, we need to add --fix for windows
golangci-lint run --timeout=20m
WARN [runner] The linter 'golint' is deprecated (since v1.41.0) due to: The repository of the linter has been archived by the owner.  Replaced by revive. 
config/decode.go:107:21      godot             Comment should end in a period
config/decode.go:112:27      godot             Comment should end in a period
config/decode.go:117:27      godot             Comment should end in a period
logger/dapr_logger.go:31:17  revive            var-declaration: should omit type string from declaration of var DaprVersion; it will be inferred from the right-hand side
logger/dapr_logger.go:71:16  exhaustivestruct  DisableTimestamp, DisableHTMLEscape, DataKey, CallerPrettyfier, PrettyPrint are missing in JSONFormatter
logger/dapr_logger.go:76:16  exhaustivestruct  ForceColors, DisableColors, ForceQuote, DisableQuote, EnvironmentOverrideColors, DisableTimestamp, FullTimestamp, DisableSorting, SortingFunc, DisableLevelTruncation, PadLevelText, QuoteEmptyFields, CallerPrettyfier are missing in TextFormatter
config/decode.go:86:3        forcetypeassert   type assertion must be checked
config/decode.go:89:3        forcetypeassert   type assertion must be checked
```

检查对比了一下 dapr CI 中是怎么进行 lint 检测的，发现原来 dapr ci 中用的是 golangci-lint 'v1.31' 版本：

```bash
Requested golangci-lint 'v1.31', using 'v1.31.0', calculation took 119ms
Installing golangci-lint v1.31.0...
Downloading https://github.com/golangci/golangci-lint/releases/download/v1.31.0/golangci-lint-1.31.0-linux-amd64.tar.gz ...
/usr/bin/tar xz --warning=no-unknown-keyword -C /home/runner -f /home/runner/work/_temp/0f22eea4-f347-44ce-af0e-a2575ae885ef
Installed golangci-lint into /home/runner/golangci-lint-1.31.0-linux-amd64/golangci-lint in 1106ms
```

删除安装的1.43版本，重新安装1.31版本：

```bash
$ rm /home/sky/work/soft/gopath/bin/golangci-lint
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.31.0
golangci/golangci-lint info checking GitHub for tag 'v1.31.0'
golangci/golangci-lint info found version: 1.31.0 for v1.31.0/linux/amd64
golangci/golangci-lint info installed /home/sky/work/soft/gopath/bin/golangci-lint
$ golangci-lint --version
golangci-lint has version 1.31.0 built from 3d6d0e7 on 2020-09-07T15:14:41Z
```

再次执行，这次结果就和 dapr ci 对应上了。

```bash
$ make lint
# Due to https://github.com/golangci/golangci-lint/issues/580, we need to add --fix for windows
golangci-lint run --timeout=20m
```

{{% alert title="切记" color="warning" %}}
 golangci-lint 一定要安装 1.31 版本！
{{% /alert %}}

#### In m1 macbook

在 m1 macbook 上， 由于  1.31 版本发布较早，没有提供对 m1 （也就是darwin-arm64）的支持，因此上面的脚本在运行时并不能自动下载安装：

```bash
$ curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.31.0
golangci/golangci-lint info checking GitHub for tag 'v1.31.0'
golangci/golangci-lint info found version: 1.31.0 for v1.31.0/darwin/arm64
```

解决方法就是开启 Rosetta 来兼容 intel 芯片：

[If you need to install Rosetta on your Mac - Apple Support](https://support.apple.com/en-us/HT211861)

通常在第一次运行基于inte芯片构建的应用时会提示。

然后手工下载 v1.31.0 的 darwin-amd64 二进制文件：

- https://github.com/golangci/golangci-lint/releases/tag/v1.31.0
- https://github.com/golangci/golangci-lint/releases/download/v1.31.0/golangci-lint-1.31.0-darwin-amd64.tar.gz

将解压缩得到的 golangci-lint 文件移动到 gopath 下的 bin 目录即可。



### Target: go.mod  

```makefile
################################################################################
# Target: go.mod                                                               #
################################################################################
.PHONY: go.mod
go.mod:
	go mod tidy
```

这个好像没啥好说的，我个人其实更习惯直接执行 `go mod tidy` 命令。

### Target: check-diff 

```makefile
################################################################################
# Target: check-diff                                                           #
################################################################################
.PHONY: check-diff
check-diff:
	git diff --exit-code ./go.mod # check no changes
```

调用 git diff 命令检查 go.mod 文件是否有改动。
