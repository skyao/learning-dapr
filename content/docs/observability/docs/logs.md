---
title: "可观测性文档中的日志"
linkTitle: "日志"
weight: 916
date: 2021-01-31
description: >
  Dapr可观测性文档中的日志
---


> 内容节选自：https://docs.dapr.io/developing-applications/building-blocks/observability/logs/

Dapr以纯文本或JSON格式生成结构化日志到stdout。默认情况下，所有的Dapr进程（运行时和系统服务）都以纯文本的形式写入控制台。要启用JSON格式的日志，需要在运行Dapr进程时添加-log-as-json命令标志。

如果要使用搜索引擎（如 Elastic Search 或 Azure Monitor）搜索日志，建议使用 JSON 格式的日志，日志收集器和搜索引擎可以使用内置的 JSON 分析器进行解析。

### Log schema

Dapr根据以下模式生成日志：

| 字段     | 描述                              | 例子                       |
| -------- | --------------------------------- | -------------------------- |
| time     | ISO8601 Timestamp                 | `2011-10-05T14:48:00.000Z` |
| level    | Log Level (info/warn/debug/error) | `info`                     |
| type     | Log Type                          | `log`                      |
| msg      | Log Message                       | `hello dapr!`              |
| scope    | Logging Scope                     | `dapr.runtime`             |
| instance | Container Name                    | `dapr-pod-xxxxx`           |
| app_id   | Dapr App ID                       | `dapr-app`                 |
| ver      | Dapr Runtime Version              | `0.5.0`                    |

### 纯文本和JSON格式的日志

- 纯文本日志示例

```bash
time="2020-03-11T17:08:48.303776-07:00" level=info msg="starting Dapr Runtime -- version 0.5.0-rc.2 -- commit v0.3.0-rc.0-155-g5dfcf2e" instance=dapr-pod-xxxx scope=dapr.runtime type=log ver=0.5.0-rc.2
time="2020-03-11T17:08:48.303913-07:00" level=info msg="log level set to: info" instance=dapr-pod-xxxx scope=dapr.runtime type=log ver=0.5.0-rc.2
```

- JSON格式日志示例

```json
{"instance":"dapr-pod-xxxx","level":"info","msg":"starting Dapr Runtime -- version 0.5.0-rc.2 -- commit v0.3.0-rc.0-155-g5dfcf2e","scope":"dapr.runtime","time":"2020-03-11T17:09:45.788005Z","type":"log","ver":"0.5.0-rc.2"}
{"instance":"dapr-pod-xxxx","level":"info","msg":"log level set to: info","scope":"dapr.runtime","time":"2020-03-11T17:09:45.788075Z","type":"log","ver":"0.5.0-rc.2"}
```

### 配置纯文本或JSON格式的日志

Dapr支持纯文本和JSON格式的日志。默认的格式是纯文本。如果您想使用纯文本与搜索引擎，您不需要更改任何配置选项。

要使用JSON格式的日志，您需要在安装Dapr和部署应用程序时添加额外的配置。建议使用JSON格式的日志，因为大多数日志收集器和搜索引擎可以通过内置的解析器更容易地解析JSON。

### 在Kubernetes中配置日志格式

以下步骤描述了如何为Kubernetes配置JSON格式的日志。

#### 使用Helm chart将Dapr安装到集群中

您可以通过在Helm命令中添加`--set global.logAsJson=true`选项为Dapr系统服务启用JSON格式的日志。

```bash
helm install dapr dapr/dapr --namespace dapr-system --set global.logAsJson=true
```

#### 为 Dapr Sidecar启用 JSON 格式的日志。

您可以通过在 deployment 中添加 `dapr.io/log-as-json: "true " `注解，在Dapr sidecars-injector服务激活的Dapr sidecars中启用JSON格式的日志。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pythonapp
  namespace: default
  labels:
    app: python
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python
  template:
    metadata:
      labels:
        app: python
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "pythonapp"
        dapr.io/log-as-json: "true"
...
```

### 日志收集器

如果你在Kubernetes集群中运行Dapr，Fluentd是一个流行的容器日志收集器。你可以使用Fluentd和json解析器插件来解析Dapr JSON格式的日志。本攻略介绍了如何在集群中配置Fleuntd。

如果你使用Azure Kubernetes服务，你可以使用默认的OMS代理与Azure Monitor收集日志，而不需要安装Fluentd。

### 搜索引擎

如果你使用Fluentd，我们建议使用Elastic Search和Kibana。这篇攻略介绍了如何在Kubernetes集群中设置Elastic Search和Kibana。

如果你使用的是Azure Kubernetes服务，你可以使用Azure监控容器，而无需安装任何额外的监控工具。另请阅读如何为容器启用Azure Monitor。