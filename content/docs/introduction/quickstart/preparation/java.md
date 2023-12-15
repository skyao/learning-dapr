---
title: "安装jdk"
linkTitle: "Java"
weight: 40
date: 2021-01-29
description: >
  为运行基于 java 的 quickstart 做准备
---



## 要求

docker 的 quickstart 对 java 的要求是 jdk 11

为了方便管理不同版本的 jdk, 推荐使用 sdkman

## 安装 sdkman

参考： https://skyao.io/learning-ubuntu-server/docs/development/common/sdkman.html

### ubuntu 20.04

```bash
sudo apt install unzip zip
curl -s "https://get.sdkman.io" | bash
```

## 安装 jdk 11

### ubuntu 20.04

执行

```bash
sdk list java
```

可以看到各种可选择的 jdk ，选择 `11.0.20-zulu` ：

```bash
sdk install java 11.0.20-zulu
```



## 安装 maven

### ubuntu20.04

```bash
sudo apt install maven
```

