---
type: docs
title: "version的源码学习"
linkTitle: "version"
weight: 99
date: 2021-02-27
description: >
  Dapr version package的源码学习
---

### 代码实现

version 的代码超级简单，就一个 version.go，内容也只有一点点：

```go
// Values for these are injected by the build.
var (
   version = "edge"
   commit  string
)

// Version returns the Dapr version. This is either a semantic version
// number or else, in the case of unreleased code, the string "edge".
func Version() string {
   return version
}

// Commit returns the git commit SHA for the code that Dapr was built from.
func Commit() string {
   return commit
}
```

- version：要不就是语义话版本，比如 `1.0.0` 这种，要不就是 `edge` 表示未发布的代码
- commit：build的时候的 git commit

### 如何注入

>  Values for these are injected by the build.

那是怎么注入的呢？ Build 总不能调用代码，而且这两个值也是private。

Dapr 下的 Makefile 文件中：

```bash
# git rev-list -1 HEAD 得到的 git commit 的 hash 值
# 如：63147334aa246d76f9f65708c257460567a1cff4
GIT_COMMIT  = $(shell git rev-list -1 HEAD)
# git describe --always --abbrev=7 --dirty 得到的是版本信息
# 如：v1.0.0-rc.4-5-g6314733
GIT_VERSION = $(shell git describe --always --abbrev=7 --dirty)

ifdef REL_VERSION
   DAPR_VERSION := $(REL_VERSION)
else
   DAPR_VERSION := edge
endif

BASE_PACKAGE_NAME := github.com/dapr/dapr

DEFAULT_LDFLAGS:=-X $(BASE_PACKAGE_NAME)/pkg/version.commit=$(GIT_VERSION) -X $(BASE_PACKAGE_NAME)/pkg/version.version=$(DAPR_VERSION)

ifeq ($(origin DEBUG), undefined)
  BUILDTYPE_DIR:=release
  LDFLAGS:="$(DEFAULT_LDFLAGS) -s -w"
else ifeq ($(DEBUG),0)
  BUILDTYPE_DIR:=release
  LDFLAGS:="$(DEFAULT_LDFLAGS) -s -w"
else
  BUILDTYPE_DIR:=debug
  GCFLAGS:=-gcflags="all=-N -l"
  LDFLAGS:="$(DEFAULT_LDFLAGS)"
  $(info Build with debugger information)
endif

define genBinariesForTarget
.PHONY: $(5)/$(1)
$(5)/$(1):
	CGO_ENABLED=$(CGO) GOOS=$(3) GOARCH=$(4) go build $(GCFLAGS) -ldflags=$(LDFLAGS) \
	-o $(5)/$(1) $(2)/;
endef

```

TODO：没看懂，有时间详细研究一下这个makefile。