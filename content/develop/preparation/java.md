---
title: "安装Java"
linkTitle: "Java"
weight: 21
date: 2022-01-24
description: >
  安装jdk/maven等Java开发工具
---

如果需要开发 dapr java-sdk，则需要安装 Java SDK。

为了方便在多个 JDK 版本中切换，建议采用 sdkman 来管理 JDK 版本。

## 安装 sdkman

参考：

- https://skyao.io/learning-ubuntu-server/docs/development/common/sdkman.html
- https://skyao.io/learning-macos/docs/programing/common/sdkman.html

## 判断需要的 jdk 版本

参考java sdk 项目的 github workflow 文件设定：

https://github.com/dapr/java-sdk/blob/master/.github/workflows/build.yml

对 java 要求都是17，也就是 Java 17

```yaml
      matrix:
        java: [ 17 ]
        spring-boot-version: [ 3.0.13 ]
```

## 安装 jdk 

```bash
sdk list java
sdk install java 17.0.10-zulu
```

验证：

```bash
java --version
openjdk 17.0.10 2024-01-16 LTS
OpenJDK Runtime Environment Zulu17.48+15-CA (build 17.0.10+7-LTS)
OpenJDK 64-Bit Server VM Zulu17.48+15-CA (build 17.0.10+7-LTS, mixed mode, sharing)
```

## 安装 maven

```bash
sdk install maven 3.9.6
```

验证：

```bash
$ mvn --version

Apache Maven 3.9.6 (bc0240f3c744dd6b6ec2920b3cd08dcc295161ae)
Maven home: /home/sky/.sdkman/candidates/maven/current
Java version: 17.0.10, vendor: Azul Systems, Inc., runtime: /home/sky/.sdkman/candidates/java/17.0.10-zulu
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "6.1.0-17-amd64", arch: "amd64", family: "unix"
```