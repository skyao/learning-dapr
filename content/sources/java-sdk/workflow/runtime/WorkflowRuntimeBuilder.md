---
title: "WorkflowRuntimeBuilder实现"
linkTitle: "WorkflowRuntimeBuilder"
weight: 20
date: 2022-07-24
description: >
  WorkflowRuntime的代码实现
---

### 类定义

WorkflowRuntimeBuilder 用来构建 WorkflowRuntime，类似 WorkflowRuntime 只是简单封装了 durabletask 的 DurableTaskGrpcWorker， WorkflowRuntimeBuilder 的实现也是简单封装了 durabletask 的 DurableTaskGrpcWorkerBuilder：

```java
import com.microsoft.durabletask.DurableTaskGrpcWorkerBuilder;

public class WorkflowRuntimeBuilder {
  private static volatile WorkflowRuntime instance;
  private DurableTaskGrpcWorkerBuilder builder;

  public WorkflowRuntimeBuilder() {
    this.builder = new DurableTaskGrpcWorkerBuilder().grpcChannel(NetworkUtils.buildGrpcManagedChannel());
  }
  ......
}
```

grpcChannel()的细节后面细看。

### registerWorkflow()方法

registerWorkflow() 方法注册 workflow 对象，实际代理给 DurableTaskGrpcWorkerBuilder 的 addOrchestration() 方法：

```java
  public <T extends Workflow> WorkflowRuntimeBuilder registerWorkflow(Class<T> clazz) {
    this.builder = this.builder.addOrchestration(
        new OrchestratorWrapper<>(clazz)
    );

    return this;
  }
```

### registerActivity() 方法

registerActivity() 方法注册 activity 对象，实际代理给 DurableTaskGrpcWorkerBuilder 的 addActivity() 方法：

```java
  public <T extends WorkflowActivity> void registerActivity(Class<T> clazz) {
    this.builder = this.builder.addActivity(
        new ActivityWrapper<>(clazz)
    );
  }
```

### build() 方法

build() 方法实现了一个简单的单例，只容许构建一个 WorkflowRuntime 的 instance：

```java
private static volatile WorkflowRuntime instance;  

public WorkflowRuntime build() {
    if (instance == null) {
      synchronized (WorkflowRuntime.class) {
        if (instance == null) {
          instance = new WorkflowRuntime(this.builder.build());
        }
      }
    }
    return instance;
  }
```

### grpcChannel 的构建细节

DurableTaskGrpcWorkerBuilder() 在构建时，需要设置 grpcChannel，而这个 grpcChannel 是通过 NetworkUtils.buildGrpcManagedChannel() 方法来实现的。

NetworkUtils.buildGrpcManagedChannel() 在 `sdk/src/main/java/io/dapr/utils/NetworkUtils.java` 文件中，是一个通用的网络工具类。buildGrpcManagedChannel() 方法的实现如下：

```java
  
private static final String DEFAULT_SIDECAR_IP = "127.0.0.1";
private static final Integer DEFAULT_GRPC_PORT = 50001;

public static final Property<String> SIDECAR_IP = new StringProperty(
      "dapr.sidecar.ip",
      "DAPR_SIDECAR_IP",
      DEFAULT_SIDECAR_IP);

  public static final Property<Integer> GRPC_PORT = new IntegerProperty(
      "dapr.grpc.port",
      "DAPR_GRPC_PORT",
      DEFAULT_GRPC_PORT);

  public static final Property<String> GRPC_ENDPOINT = new StringProperty(
      "dapr.grpc.endpoint",
      "DAPR_GRPC_ENDPOINT",
      null);

public static ManagedChannel buildGrpcManagedChannel() {
    // 从系统属性或者环境变量中读取 dapr sidecar 的IP
    String address = Properties.SIDECAR_IP.get();
    // 从系统属性或者环境变量中读取 dapr grpc 端口
    int port = Properties.GRPC_PORT.get();
    // 默认不用https
    boolean insecure = true;
    // 从系统属性或者环境变量中读取 dapr grpc 端点信息
    String grpcEndpoint = Properties.GRPC_ENDPOINT.get();
    if ((grpcEndpoint != null) && !grpcEndpoint.isEmpty()) {
      // 如果 dapr grpc 端点不为空，则用 grpc 端点的内容覆盖 
      URI uri = URI.create(grpcEndpoint);
      // 通过 schema 是不是 http 来判断是 http 还是 https
      insecure = uri.getScheme().equalsIgnoreCase("http");
      // grpcEndpoint 如果设置有端口则采用，没有设置则根据是 http 还是 https 来选择 80 或者 443 端口
      port = uri.getPort() > 0 ? uri.getPort() : (insecure ? 80 : 443);
      // 覆盖 dapr sidecar 的地址
      address = uri.getHost();
      if ((uri.getPath() != null) && !uri.getPath().isEmpty()) {
        address += uri.getPath();
      }
    }
    
    // 构建连接到指定地址的 grpc channel
    ManagedChannelBuilder<?> builder = ManagedChannelBuilder.forAddress(address, port)
        .userAgent(Version.getSdkVersion());
    if (insecure) {
      builder = builder.usePlaintext();
    }
    return builder.build();
  }
```

从部署来看，runtime 运行在 client 一侧的 app 应用程序内部，然后通过 durabletask 的 sdk 连接到 dapr sidecar 了，走 grpc 协议。

这个设计有点奇怪，dapr sdk 和 dapr sidecar 之间没有走标准的 dapr API，而是通过 durabletask 的 sdk 。

