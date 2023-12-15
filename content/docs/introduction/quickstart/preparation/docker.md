---
title: "安装docker"
linkTitle: "docker"
weight: 10
date: 2021-01-29
description: >
  quickstart 是运行在 docker 上的
---



## 安装 docker

参考： https://skyao.io/learning-docker/docs/installation.html

### ubuntu 20.04

参考： https://skyao.io/learning-docker/docs/installation/ubuntu.html

添加 GPG key：

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

准备用于 apt 的仓库：

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

执行 apt update :

```bash
sudo apt-get update
```

安装 20.10.21 版本：

```bash
VERSION_STRING=5:20.10.21~3-0~ubuntu-focal
sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

为了以非 root 用户使用 docker：

```bash
sudo usermod -aG docker sky
```

重新登录或者重启。
