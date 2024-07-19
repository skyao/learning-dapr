---
title: "安装golang"
linkTitle: "golang"
weight: 20
date: 2022-01-24
description: >
  安装golang
---

## 版本要求

dapr 对 golang 的版本一直要求比较高，基本上是用最新版本的 golang。

具体版本要求可以查看 dapr 下的 go.mod 中的要求，如：

https://github.com/dapr/dapr/blob/master/go.mod

```go
module github.com/dapr/dapr

go 1.21

toolchain go1.21.4
```

## 下载安装

在golang官方下载地址下载安装对应的版本即可。

https://golang.org/dl/

安装方式参考官方安装文档：

https://go.dev/doc/install

### linux 安装

```
mkdir -p ~/temp
cd ~/temp

wget https://go.dev/dl/go1.21.6.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.6.linux-amd64.tar.gz

mkdir -p ~/work/soft/gopath
```

修改 PATH：

```bash
# go
export GOPATH=$HOME/work/soft/gopath
export PATH=/usr/local/go/bin:$GOPATH/bin:$PATH
```

验证：

```bash
$ go version                               
go version go1.21.6 linux/amd64
```