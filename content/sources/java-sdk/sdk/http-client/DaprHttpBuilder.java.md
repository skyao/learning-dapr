---
title: "DaprHttpBuilder"
linkTitle: "DaprHttpBuilder"
weight: 30
date: 2021-02-24
description: >
  builder for DaprHttp，基于 okhttp
---

代码没啥特殊的，就注意一下 okhttp 的一些参数的获取。

另外 MaxRequestsPerHost 默认为5，这是一个超级大坑！

```java
private DaprHttp buildDaprHttp() {
    // 双重检查锁
    if (OK_HTTP_CLIENT == null) {
      synchronized (LOCK) {
        if (OK_HTTP_CLIENT == null) {
          OkHttpClient.Builder builder = new OkHttpClient.Builder();
          Duration readTimeout = Duration.ofSeconds(Properties.HTTP_CLIENT_READ_TIMEOUT_SECONDS.get());
          builder.readTimeout(readTimeout);

          Dispatcher dispatcher = new Dispatcher();
          dispatcher.setMaxRequests(Properties.HTTP_CLIENT_MAX_REQUESTS.get());
          //这里有一个超级大坑！
          // The maximum number of requests for each host to execute concurrently.
          // Default value is 5 in okhttp which is totally UNACCEPTABLE!
          // For sidecar case, set it the same as maxRequests.
          dispatcher.setMaxRequestsPerHost(Properties.HTTP_CLIENT_MAX_REQUESTS.get());
          builder.dispatcher(dispatcher);

          ConnectionPool pool = new ConnectionPool(Properties.HTTP_CLIENT_MAX_IDLE_CONNECTIONS.get(),
                  KEEP_ALIVE_DURATION, TimeUnit.SECONDS);
          builder.connectionPool(pool);

          OK_HTTP_CLIENT = builder.build();
        }
      }
    }

    return new DaprHttp(Properties.SIDECAR_IP.get(), Properties.HTTP_PORT.get(), OK_HTTP_CLIENT);
  }
}
```

相关的几个参数的获取：

- http read timeout：默认60秒，可以通过系统参数 dapr.http.client.maxRequests 或者环境变量 DAPR_HTTP_CLIENT_READ_TIMEOUT_SECONDS 覆盖
- http max request：默认 1024，可以通过系统参数 dapr.http.client.readTimeoutSeconds 或者环境变量 DAPR_HTTP_CLIENT_MAX_REQUESTS 覆盖
- http max idle connections: 默认 1024，可以通过系统参数 dapr.http.client.maxIdleConnections 或者环境变量 DAPR_HTTP_CLIENT_MAX_IDLE_CONNECTIONS 覆盖
- HTTP keep alive duration： hard code 为 30

对于 okhttp，还必须自动设置  http max request per host 参数，不然默认值为 5 对于 sidecar 来说完全不可用。