---
title: "Dapr的命令行参数"
linkTitle: "命令行参数"
weight: 112
date: 2021-01-29
description: >
  Dapr的命令行参数
---



| name                    | default value | 可选值                  | 说明                                                         |
| ----------------------- | ------------- | ----------------------- | ------------------------------------------------------------ |
| mode                    | standalone    | standalone / kubernetes | Runtime mode for Dapr                                        |
| dapr-http-port          | 3500          |                         | HTTP port for Dapr API to listen on                          |
| dapr-grpc-port          | 50001         |                         | gRPC port for the Dapr API to listen on                      |
| dapr-internal-grpc-port | ""            |                         | gRPC port for the Dapr Internal API to listen on<br />如果不指定，则dapr会自动获取一个随机可用的空闲端口 |
| app-port                | ""            |                         | The port the application is listening on<br />如果不设置，则不能建立dapr和应用之间的 app channel <br />也就意味着dapr 无法发送请求给应用了<br />同时dapr也无法从应用读取配置： `http://localhost:<app_port>/dapr/config`<br />注意：app的地址在代码中写死 127.0.0.1 |
| profile-port            | 7777          |                         | The port for the profile server                              |
| app-protocol            | http          | http / grpc             | Protocol for the application: grpc or http                   |
| components-path         | ""            |                         | Path for components directory. If empty, components will not be loaded. Self-hosted mode only<br />仅在dapr mode为standalone时有效 |
| config                  | ""            |                         | Path to config file, or name of a configuration object<br />被称为 global configuration<br />如果是kubernetes mode，读取k8s的配置<br />如果是standalone mode，读取配置文件<br />如果没有配置，则装载默认配置 |
| app-id                  | ""            |                         | A unique ID for Dapr. Used for Service Discovery and state   |
| control-plane-address   | ""            |                         | Address for a Dapr control plane<br />仅在dapr mode为kubernetes时有效 |
| sentry-address          | ""            |                         | Address for the Sentry CA service                            |
| placement-host-address  | ""            |                         | Address for the Dapr placement service                       |
| allowed-origins         | `*`           |                         | Allowed HTTP origins                                         |
| enable-profiling        | false         | True / false            | Enable profiling                                             |
| version                 | false         | True / false            | Prints the runtime version （然后dapr就会退出）              |
| app-max-concurrency     | -1            |                         | Controls the concurrency level when forwarding requests to user code |
| enable-mtls             | false         | True / false            | Enables automatic mTLS for daprd to daprd communication channels |
|                         |               |                         |                                                              |



1.0 之后被弃用的flag：

| name              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| placement-address | 改为 placement-host-address<br />如果同时设置，以 placement-host-address 为准 |
| max-concurrency   | 改为 app-max-concurrency<br />如果同时设置，以 app-max-concurrency 为准 |
| protocol          | 改为 app-protocol<br />如果同时设置，以 app-protocol 为准    |









-app-id hellogrpc -app-port 3000 -protocol grpc -dapr-http-port 3005 -dapr-grpc-port 52000 -placement-address localhost:50005 -components-path components





