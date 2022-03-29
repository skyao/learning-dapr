---
type: docs
title: "debug"
linkTitle: "debug"
weight: 9999
date: 2021-01-31
description: >
  debug
---



## 备用

```plantuml
title Service Invoke via HTTP
hide footbox
skinparam style strictuml
box "App-1"
participant user_code_client [
    =App-1
    ----
    client
]
participant SDK_client [
    =SDK
    ----
    client
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant SDK_server [
    =SDK
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]
end box

user_code_client -> SDK_client : Invoke\nService() 
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: HTTP API @ 3500
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
SDK_client <[#blue]-- daprd_client
user_code_client <-- SDK_client
```

### 命令

启动服务器端的命令：

```bash
cd /home/sky/work/code/dapr/java-sdk/examples

dapr run --app-id invokedemo --app-port 3000 -- java -jar target/dapr-java-sdk-examples-exec.jar io.dapr.examples.invoke.http.DemoService -p 3000 
```

启动客户端的命令：

```bash
cd /home/sky/work/code/dapr/java-sdk/examples

dapr run --app-id invokeclient -- java -jar target/dapr-java-sdk-examples-exec.jar io.dapr.examples.invoke.http.InvokeClient "message one" "message two"
```



### 服务器端日志

日志输出，备用：

```bash
dapr run --app-id invokedemo --app-port 3000 -- java -jar target/dapr-java-sdk-examples-exec.jar io.dapr.examples.invoke.http.DemoService -p 3000       
ℹ️  Starting Dapr with id invokedemo. HTTP Port: 43751. gRPC Port: 36405
INFO[0000] starting Dapr Runtime -- version edge -- commit d4e49a28786e60366685a995f1ff8cefc3b1437b  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] log level set to: info                        app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] metrics server started on :46177/             app_id=invokedemo instance=skywork scope=dapr.metrics type=log ver=edge
INFO[0000] standalone mode configured                    app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] app id: invokedemo                            app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] mTLS is disabled. Skipping certificate request and tls validation  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] local service entry announced: invokedemo -> 10.0.0.10:41005  app_id=invokedemo instance=skywork scope=dapr.contrib type=log ver=edge
INFO[0000] Initialized name resolution to mdns           app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] loading components                            app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] component loaded. name: pubsub, type: pubsub.redis/v1  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] waiting for all outstanding components to be processed  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] detected actor state store: statestore        app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] component loaded. name: statestore, type: state.redis/v1  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] all outstanding components processed          app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] gRPC proxy enabled                            app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] enabled gRPC tracing middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.grpc.api type=log ver=edge
INFO[0000] enabled gRPC metrics middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.grpc.api type=log ver=edge
INFO[0000] API gRPC server is running on port 36405      app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] enabled metrics http middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.http type=log ver=edge
INFO[0000] enabled tracing http middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.http type=log ver=edge
INFO[0000] http server is running on port 43751          app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] The request body size parameter is: 4         app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] enabled gRPC tracing middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.grpc.internal type=log ver=edge
INFO[0000] enabled gRPC metrics middleware               app_id=invokedemo instance=skywork scope=dapr.runtime.grpc.internal type=log ver=edge
INFO[0000] internal gRPC server is running on port 41005  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0000] application protocol: http. waiting on port 3000.  This will block until the app is listening on that port.  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
== APP == 
== APP ==   .   ____          _            __ _ _
== APP ==  /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
== APP == ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
== APP ==  \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
== APP ==   '  |____| .__|_| |_|_| |_\__, | / / / /
== APP ==  =========|_|==============|___/=/_/_/_/
== APP ==  :: Spring Boot ::        (v2.3.5.RELEASE)
== APP == 
== APP == 2022-03-16 17:20:55,316 {HH:mm:ss.SSS} [main] INFO  i.d.examples.invoke.http.DemoService - Starting DemoService on skywork with PID 184091 (/home/sky/work/code/dapr/java-sdk/examples/target/dapr-java-sdk-examples-exec.jar started by sky in /home/sky/work/code/dapr/java-sdk/examples)
== APP == 2022-03-16 17:20:55,318 {HH:mm:ss.SSS} [main] INFO  i.d.examples.invoke.http.DemoService - No active profile set, falling back to default profiles: default
== APP == 2022-03-16 17:20:55,406 {HH:mm:ss.SSS} [background-preinit] WARN  o.s.h.c.j.Jackson2ObjectMapperBuilder - For Jackson Kotlin classes support please add "com.fasterxml.jackson.module:jackson-module-kotlin" to the classpath
ℹ️  Updating metadata for app command: java -jar target/dapr-java-sdk-examples-exec.jar io.dapr.examples.invoke.http.DemoService -p 3000
✅  You're up and running! Both Dapr and your app logs will appear here.

== APP == 2022-03-16 17:20:56,055 {HH:mm:ss.SSS} [main] INFO  o.s.b.w.e.tomcat.TomcatWebServer - Tomcat initialized with port(s): 3000 (http)
== APP == 2022-03-16 17:20:56,064 {HH:mm:ss.SSS} [main] INFO  o.a.coyote.http11.Http11NioProtocol - Initializing ProtocolHandler ["http-nio-3000"]
== APP == 2022-03-16 17:20:56,064 {HH:mm:ss.SSS} [main] INFO  o.a.catalina.core.StandardService - Starting service [Tomcat]
== APP == 2022-03-16 17:20:56,064 {HH:mm:ss.SSS} [main] INFO  o.a.catalina.core.StandardEngine - Starting Servlet engine: [Apache Tomcat/9.0.39]
== APP == 2022-03-16 17:20:56,117 {HH:mm:ss.SSS} [main] INFO  o.a.c.c.C.[Tomcat].[localhost].[/] - Initializing Spring embedded WebApplicationContext
== APP == 2022-03-16 17:20:56,117 {HH:mm:ss.SSS} [main] INFO  o.s.b.w.s.c.ServletWebServerApplicationContext - Root WebApplicationContext: initialization completed in 752 ms
== APP == 2022-03-16 17:20:56,798 {HH:mm:ss.SSS} [main] INFO  o.a.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-3000"]
INFO[0002] application discovered on port 3000           app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
== APP == 2022-03-16 17:20:56,809 {HH:mm:ss.SSS} [main] INFO  o.s.b.w.e.tomcat.TomcatWebServer - Tomcat started on port(s): 3000 (http) with context path ''
== APP == 2022-03-16 17:20:56,827 {HH:mm:ss.SSS} [main] INFO  i.d.examples.invoke.http.DemoService - Started DemoService in 1.823 seconds (JVM running for 2.292)
== APP == 2022-03-16 17:20:56,897 {HH:mm:ss.SSS} [http-nio-3000-exec-2] INFO  o.a.c.c.C.[Tomcat].[localhost].[/] - Initializing Spring DispatcherServlet 'dispatcherServlet'
== APP == 2022-03-16 17:20:56,897 {HH:mm:ss.SSS} [http-nio-3000-exec-2] INFO  o.s.web.servlet.DispatcherServlet - Initializing Servlet 'dispatcherServlet'
== APP == 2022-03-16 17:20:56,901 {HH:mm:ss.SSS} [http-nio-3000-exec-2] INFO  o.s.web.servlet.DispatcherServlet - Completed initialization in 4 ms
INFO[0002] application configuration loaded              app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0002] actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s  app_id=invokedemo instance=skywork scope=dapr.runtime.actor type=log ver=edge
INFO[0002] placement tables updated, version: 0          app_id=invokedemo instance=skywork scope=dapr.runtime.actor.internal.placement type=log ver=edge
INFO[0002] app is subscribed to the following topics: [testingtopic] through pubsub=messagebus  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
INFO[0002] dapr initialized. Status: Running. Init Elapsed 2399.544519ms  app_id=invokedemo instance=skywork scope=dapr.runtime type=log ver=edge
```

有用的信息：

- API gRPC server port: 36405 
- http server port: 43751 
- internal gRPC server port: 41005
- application protocol: http. 
- application port: 3000

### 客户端日志



## 测试场景

### 完整路径： app client 到 app server



```bash
rm /home/sky/.dapr/bin/daprd
cp /home/sky/work/code/dapr/dapr/dist/linux_amd64/release/daprd ~/.dapr/bin/
```



## 

```plantuml
title Service Invoke by default
hide footbox
skinparam style strictuml
box "invokeclient"
participant user_code_client [
    =invokeclient
    ----
    java
]
participant SDK_client [
    =SDK
    ----
    java
]
end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "invokeDemo"
participant SDK_server [
    =SDK
    ----
    java
]
participant user_code_server [
    =invokeDemo
    ----
    java
]
end box

user_code_client -> SDK_client : Invoke\nService() 
SDK_client -[#blue]> daprd_client : HTTP (localhost)
note right: issue2: fasthttp concurrency limitation (5??)
|||
daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: issue1: mDNS 
|||
daprd_server -> SDK_server : HTTP (localhost)
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <-- SDK_server
daprd_client <-- daprd_server
SDK_client <-- daprd_client
user_code_client <-- SDK_client
```

### 



单服务器： 直接连 app server



单服务器： 连 app server 的 daprd




```plantuml
title Service Invoke via HTTP
hide footbox
skinparam style strictuml
box "App-1"

end box
participant daprd_client [
    =daprd
    ----
    client
]
participant daprd_server [
    =daprd
    ----
    server
]

box "App-2"
participant SDK_server [
    =SDK
    ----
    server
]
participant user_code_server [
    =App-2
    ----
    server
]
end box


daprd_client -[#red]> daprd_server : gRPC (remote call)
note right: internal API @ ramdon free port
|||
daprd_server -[#blue]> SDK_server : gRPC (localhost)
note right: 50001
SDK_server -> user_code_server : 

SDK_server <-- user_code_server
daprd_server <[#blue]-- SDK_server
daprd_client <[#red]-- daprd_server
```





This is a very tricky bug...... I sent so many times to debug it. 



As shown in above picture, by default, if no specified configuration, the service invoke flow will like this. 

- In the java SDK, it will use HTTP to connect to daprd via dapr HTTP API
- Between two daprd, the protocol will be gRPC (in fact only gRPC support), and mDNS will be used as Name Resolver
- Daprd in server side will use HTTP to send request to the app (which is invokeDemo in the picture)

Now I have found two issues:

### mDNS

**mDNS Name Resolver will be very slow (response in more than 10 seconds) in multi-thread.** 

In daprd, we will call the name resolver for each request without cache or lock:

https://github.com/dapr/dapr/blob/3fbb9326503a64f998c5b2e65afcd12cf89d7925/pkg/messaging/direct_messaging.go#L246

```go
func (d *directMessaging) getRemoteApp(appID string) (remoteApp, error) {
	id, namespace, err := d.requestAppIDAndNamespace(appID)
	if err != nil {
		return remoteApp{}, err
	}
	
	request := nr.ResolveRequest{ID: id, Namespace: namespace, Port: d.grpcPort}
	address, err := d.resolver.ResolveID(request)
	if err != nil {
		return remoteApp{}, err
	}
	
	return remoteApp{
		namespace: namespace,
		id:        id,
		address:   address,
	}, nil
}
```

I tried to use a sync.once (only for verification since in the test case I only have one app to look up), it work well.

### fasthttp

I added some log to see where the request are blocked. 

From the log I saw that:

1. all the 50 requests will be sent by java sdk to daprd
2. but in daprd log of HTTP API, it seems that by default there are only 5 requests accepted and served. Others are in the queue and wait. 

I will try to find what happened in fasthttp. 