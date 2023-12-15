---
title: "安装 dapr cli"
linkTitle: "安装 dapr cli"
weight: 20
date: 2022-04-09
description: >
  安装Dapr并进行初始化
---

## 安装 dapr CLI

参考： https://docs.dapr.io/getting-started/install-dapr-cli/ 

### ubuntu 20.04

```bash
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
```

输出为：

```bash
Getting the latest Dapr CLI...
Your system is linux_amd64
Installing Dapr CLI...

Installing v1.11.0 Dapr CLI...
Downloading https://github.com/dapr/cli/releases/download/v1.11.0/dapr_linux_amd64.tar.gz ...
dapr installed into /usr/local/bin successfully.
CLI version: 1.11.0 
Runtime version: 1.11.3

To get started with Dapr, please visit https://docs.dapr.io/getting-started/
```

安装后检查：

```bash
dapr -h
```



## 初始化 dapr

参考： https://docs.dapr.io/getting-started/install-dapr-selfhost/

### ubuntu 20.04

```bash
dapr init
```

输出为：

```bash
⌛  Making the jump to hyperspace...
ℹ️  Container images will be pulled from Docker Hub
ℹ️  Installing runtime version 1.11.3
←  Downloading binaries and setting up components...
Dapr runtime installed to /home/sky/.dapr/bin, you may run the following to add it to your path if you want to run daprd directly:
    export PATH=$PATH:/home/sky/.dapr/bin
✅  Downloading binaries and setting up components...
✅  Downloaded binaries and completed components set up.
ℹ️  daprd binary has been installed to /home/sky/.dapr/bin.
ℹ️  dapr_placement container is running.
ℹ️  dapr_redis container is running.
ℹ️  dapr_zipkin container is running.
ℹ️  Use `docker ps` to check running containers.
✅  Success! Dapr is up and running. To get started, go here: https://aka.ms/dapr-getting-started
```

此时执行：

```bash
docker ps
```

可以看到当前和 dapr 相关的几个容器正在运行：

```bash

CONTAINER ID   IMAGE                COMMAND                  CREATED              STATUS                   PORTS                                                 NAMES
bf674f47c1b9   daprio/dapr:1.11.3   "./placement"            About a minute ago   Up About a minute        0.0.0.0:50005->50005/tcp, :::50005->50005/tcp         dapr_placement
b26edce4fd09   openzipkin/zipkin    "start-zipkin"           6 hours ago          Up 2 minutes (healthy)   9410/tcp, 0.0.0.0:9411->9411/tcp, :::9411->9411/tcp   dapr_zipkin
0bb3b7b8e9dc   redis:6              "docker-entrypoint.s…"   6 hours ago          Up 2 minutes             0.0.0.0:6379->6379/tcp, :::6379->6379/tcp             dapr_redis
```

