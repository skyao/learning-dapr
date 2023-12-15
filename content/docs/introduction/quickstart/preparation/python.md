---
title: "安装pythod"
linkTitle: "pythod"
weight: 30
date: 2021-01-29
description: >
  为运行基于 pythod 的 quickstart 做准备
---



## 安装 pythod

docker 的 quickstart 要求 quickstart 3.7 版本及以上

### ubuntu 20.04

默认自带 pythod 3.8, 可以直接使用：

```bash
$ which python3
/usr/bin/python3
```

但奇怪的是默认不自带pip3：

```bash
$ pip3 install -r requirements.txt

zsh: command not found: pip3
```

因此需要手工安装 python3-pip ：

```bash
sudo apt install -y python3-pip
```

