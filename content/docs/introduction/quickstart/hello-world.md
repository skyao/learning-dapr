---
title: "Hello World"
linkTitle: "Hello World"
weight: 122
date: 2021-01-29
description: >
  Dapr的Hello World
---

参考官方文档

https://github.com/dapr/quickstarts/tree/master/hello-world

## 安装dapr

### 准备工作

按照 Prerequisites 的要求安装：

- docker
	
- ubuntu 安装方法：https://docs.docker.com/engine/install/ubuntu/
	
	- mac 安装方法：`brew cask install docker`  (执行前先开启一下代理，否则速度很慢)，完成后启动docker程序会继续安装。
	
- nodejs

  ```bash
  # ubuntu下
  sudo apt update
  sudo apt install nodejs npm
  
  # mac下
  brew install nodejs
  ```

- python3.8： 

  - ubuntu 20.04自带
  - macos自带

### 安装并初始化dapr

https://github.com/dapr/docs/blob/master/getting-started/environment-setup.md#environment-setup

按照文档指导可以在本地跑起来dapr和两个应用 nodejs app 和 pythonapp。

## 运行应用和dapr

启动 nodeapp：

```bash
$ dapr run --app-id nodeapp --app-port 3000 --port 3500 node app.js
ℹ️  Starting Dapr with id nodeapp. HTTP Port: 3500. gRPC Port: 41313
== DAPR == time="2020-08-13T03:01:27.401563329Z" level=info msg="starting Dapr Runtime -- version 0.9.0 -- commit 6babe85-dirty" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.401592638Z" level=info msg="log level set to: info" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.403754652Z" level=info msg="metrics server started on :45301/" app_id=nodeapp instance=server-22 scope=dapr.metrics type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.40608922Z" level=info msg="standalone mode configured" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406101799Z" level=info msg="app id: nodeapp" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406109449Z" level=info msg="mTLS is disabled. Skipping certificate request and tls validation" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406491465Z" level=info msg="found component zipkin (exporters.zipkin)" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406815324Z" level=info msg="found component pubsub (pubsub.redis)" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406827704Z" level=info msg="found component statestore (state.redis)" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.406836693Z" level=info msg="application protocol: http. waiting on port 3000.  This will block until the app is listening on that port." app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== APP == Node App listening on port 3000!

== DAPR == time="2020-08-13T03:01:27.6668991Z" level=info msg="application discovered on port 3000" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.676719603Z" level=info msg="application configuration loaded" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.683106449Z" level=info msg="local service entry announced: nodeapp -> 192.168.0.22:40147" app_id=nodeapp instance=server-22 scope=dapr.contrib type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.683134018Z" level=info msg="Initialized name resolution to standalone" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684083474Z" level=info msg="actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684125533Z" level=info msg="enabled monitoring middleware" app_id=nodeapp instance=server-22 scope=dapr.runtime.grpc.api type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684147342Z" level=info msg="API gRPC server is running on port 41313" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684159811Z" level=info msg="enabled monitoring middleware" app_id=nodeapp instance=server-22 scope=dapr.runtime.grpc.internal type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.68419098Z" level=info msg="internal gRPC server is running on port 40147" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.68447814Z" level=info msg="enabled cors http middleware" app_id=nodeapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684526578Z" level=info msg="enabled metrics http middleware" app_id=nodeapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.684532368Z" level=info msg="enabled tracing http middleware" app_id=nodeapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.685153897Z" level=info msg="http server is running on port 3500" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.685141747Z" level=info msg="starting connection attempt to placement service at localhost:50005" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.68675419Z" level=info msg="dapr initialized. Status: Running. Init Elapsed 280.688959ms" app_id=nodeapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.690870475Z" level=info msg="established connection to placement service at localhost:50005" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.695620718Z" level=info msg="placement order received: lock" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.695656467Z" level=info msg="placement order received: update" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.695664007Z" level=info msg="actors: placement tables updated" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T03:01:27.695696185Z" level=info msg="placement order received: unlock" app_id=nodeapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

ℹ️  Updating metadata for app command: node app.js
✅  You're up and running! Both Dapr and your app logs will appear here.
```

启动 pythonapp：

```bash
$ dapr run --app-id pythonapp python3 app.py
ℹ️  Starting Dapr with id pythonapp. HTTP Port: 36919. gRPC Port: 45173
== APP == HTTPConnectionPool(host='localhost', port=36919): Max retries exceeded with url: /v1.0/invoke/nodeapp/method/neworder (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f1bd71669d0>: Failed to establish a new connection: [Errno 111] Connection refused'))

== DAPR == time="2020-08-13T06:46:56.72406294Z" level=info msg="starting Dapr Runtime -- version 0.9.0 -- commit 6babe85-dirty" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.72409239Z" level=info msg="log level set to: info" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.72418481Z" level=info msg="metrics server started on :46033/" app_id=pythonapp instance=server-22 scope=dapr.metrics type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724529889Z" level=info msg="standalone mode configured" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724553109Z" level=info msg="app id: pythonapp" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724569429Z" level=info msg="mTLS is disabled. Skipping certificate request and tls validation" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724949349Z" level=info msg="found component zipkin (exporters.zipkin)" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724973299Z" level=info msg="found component pubsub (pubsub.redis)" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.724978979Z" level=info msg="found component statestore (state.redis)" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726339857Z" level=info msg="local service entry announced: pythonapp -> 192.168.0.22:41923" app_id=pythonapp instance=server-22 scope=dapr.contrib type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726362247Z" level=info msg="Initialized name resolution to standalone" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726380477Z" level=error msg="failed to init input bindings: app channel not initialized" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726423417Z" level=info msg="actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726447457Z" level=info msg="enabled monitoring middleware" app_id=pythonapp instance=server-22 scope=dapr.runtime.grpc.api type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726464967Z" level=info msg="API gRPC server is running on port 45173" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726585237Z" level=info msg="enabled monitoring middleware" app_id=pythonapp instance=server-22 scope=dapr.runtime.grpc.internal type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726699017Z" level=info msg="internal gRPC server is running on port 41923" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726787137Z" level=info msg="enabled cors http middleware" app_id=pythonapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726817007Z" level=info msg="enabled metrics http middleware" app_id=pythonapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726821527Z" level=info msg="enabled tracing http middleware" app_id=pythonapp instance=server-22 scope=dapr.runtime.http type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726828177Z" level=info msg="http server is running on port 36919" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726836537Z" level=info msg="dapr initialized. Status: Running. Init Elapsed 2.306748ms" app_id=pythonapp instance=server-22 scope=dapr.runtime type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.726891147Z" level=info msg="starting connection attempt to placement service at localhost:50005" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.727357086Z" level=info msg="established connection to placement service at localhost:50005" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.727944425Z" level=info msg="placement order received: lock" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.727973365Z" level=info msg="placement order received: update" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.727980025Z" level=info msg="actors: placement tables updated" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

== DAPR == time="2020-08-13T06:46:56.727986475Z" level=info msg="placement order received: unlock" app_id=pythonapp instance=server-22 scope=dapr.runtime.actor type=log ver=0.9.0

ℹ️  Updating metadata for app command: python3 app.py
✅  You're up and running! Both Dapr and your app logs will appear here.
```





## 观察

### 进程信息

dapr 的组件以容器的方式运行，包括 zipkin / dapr(placement) / redis 三个容器：

```bash
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                 PORTS                              NAMES
8c5fd90abf99        openzipkin/zipkin   "/bin/sh -c /zipkin/…"   46 hours ago        Up 4 hours (healthy)   9410/tcp, 0.0.0.0:9411->9411/tcp   dapr_zipkin
f9edd8935e51        daprio/dapr         "./placement"            46 hours ago        Up 4 hours             0.0.0.0:50005->50005/tcp           dapr_placement
70c1865d94b2        redis               "docker-entrypoint.s…"   46 hours ago        Up 4 hours             0.0.0.0:6379->6379/tcp             dapr_redis
```

对象的docker镜像情况：

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
openzipkin/zipkin   latest              13a9766380f5        7 days ago          157MB
redis               latest              1319b1eaa0b7        8 days ago          104MB
daprio/dapr         latest              775fe3438f2b        3 weeks ago         189MB
```



node app 和 nodeapp 的 daprd（parent 进程是pythonapp）：

```bash
$ dapr run --app-id nodeapp --app-port 3000 --port 3500 node app.js

$ ps -ef | grep nodeapp
sky        81337   65485  0 06:45 pts/0    00:00:00 dapr run --app-id nodeapp --app-port 3000 --port 3500 node app.js
sky        81354   81337  0 06:45 pts/0    00:00:00 daprd --app-id nodeapp --dapr-http-port 3500 --dapr-grpc-port 33575 --log-level info --max-concurrency -1 --protocol http --metrics-port 35191 --components-path /home/sky/.dapr/components --app-port 3000 --placement-address localhost:50005 --config /home/sky/.dapr/config.yaml
```

pythonapp 和 pythonapp 的 daprd（parent 进程是pythonapp）：

```bash
$ dapr run --app-id pythonapp python3 app.py
$ ps -ef | grep pythonapp
sky        81698   80792  0 06:46 pts/3    00:00:00 dapr run --app-id pythonapp python3 app.py
sky        81715   81698  0 06:46 pts/3    00:00:00 daprd --app-id pythonapp --dapr-http-port 36919 --dapr-grpc-port 45173 --log-level info --max-concurrency -1 --protocol http --metrics-port 46033 --components-path /home/sky/.dapr/components --placement-address localhost:50005 --config /home/sky/.dapr/config.yaml
```

注意 daprd 启动了两个进程，分别服务于 pythonapp 和 nodeapp： 

```bash
$ ps -ef | grep daprd
sky        81354   81337  0 06:45 pts/0    00:00:00 daprd --app-id nodeapp --dapr-http-port 3500 --dapr-grpc-port 33575 --log-level info --max-concurrency -1 --protocol http --metrics-port 35191 --components-path /home/sky/.dapr/components --app-port 3000 --placement-address localhost:50005 --config /home/sky/.dapr/config.yaml
sky        81715   81698  0 06:46 pts/3    00:00:00 daprd --app-id pythonapp --dapr-http-port 36919 --dapr-grpc-port 45173 --log-level info --max-concurrency -1 --protocol http --metrics-port 46033 --components-path /home/sky/.dapr/components --placement-address localhost:50005 --config /home/sky/.dapr/config.yaml
```

为了避免端口冲突，两个 dapr 进程的各种端口是不一样的。

### dapr配置信息

```bash
$ cd ~/.dapr/
$ ls -lah
total 80M
drwxrwxrwx 3 sky sky 4.0K Aug 13 07:02 .
drwxr-xr-x 9 sky sky 4.0K Aug 13 07:02 ..
drwxrwxrwx 2 sky sky 4.0K Aug 13 07:00 components
-rw-r--r-- 1 sky sky  117 Aug 11 08:33 config.yaml
-rwxr-xr-x 1 sky sky  80M Aug 11 08:34 daprd
```

其中 daprd 是一个 80M 大小的可执行文件。

config.yaml 文件的内容：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  tracing:
    samplingRate: "1"
```

/home/sky/.dapr/components 下的文件有：

```bash
$ ls /home/sky/.dapr/components
pubsub.yaml  statestore.yaml  zipkin.yaml
```

其中 pubsub.yaml 文件的内容为：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
```

state store.xml 的内容为：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

zipkin.xml 的内容为：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: zipkin
spec:
  type: exporters.zipkin
  metadata:
  - name: enabled
    value: "true"
  - name: exporterAddress
    value: http://localhost:9411/api/v2/spans
```







