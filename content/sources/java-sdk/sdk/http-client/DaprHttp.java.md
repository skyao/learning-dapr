---
title: "DaprHttp"
linkTitle: "DaprHttp"
weight: 20
date: 2021-02-24
description: >
  Dapr HTTP 的 okhttp3 + jackson 实现
---

### 常量定义

```java
  public static final String API_VERSION = "v1.0";

  public static final String ALPHA_1_API_VERSION = "v1.0-alpha1";

  private static final String HEADER_DAPR_REQUEST_ID = "X-DaprRequestId";

  private static final String DEFAULT_HTTP_SCHEME = "http";

  private static final Set<String> ALLOWED_CONTEXT_IN_HEADERS =
      Collections.unmodifiableSet(new HashSet<>(Arrays.asList("grpc-trace-bin", "traceparent", "tracestate")));
```

HTTP 方法定义：

```java
  public enum HttpMethods {
    NONE,
    GET,
    PUT,
    POST,
    DELETE,
    HEAD,
    CONNECT,
    OPTIONS,
    TRACE
  }
  ```

### 基本类定义

```java
  public static class Response {
    private byte[] body;
    private Map<String, String> headers;
    private int statusCode;
    ......
  }
```

## DaprHttp 类定义

```java
  private final OkHttpClient httpClient;
  private final int port;
  private final String hostname;

  DaprHttp(String hostname, int port, OkHttpClient httpClient) {
    this.hostname = hostname;
    this.port = port;
    this.httpClient = httpClient;
  }
```


### invokeApi() 方法实现

这个方法有多个重载，最终的实现如下，用来执行http调用请求：

```java
  /**
   * 调用API，返回文本格式有效载荷。
   *
   * @param method        HTTP method.
   * @param pathSegments  Array of path segments (/a/b/c -> ["a", "b", "c"]).
   * @param urlParameters Parameters in the URL
   * @param content       payload to be posted.
   * @param headers       HTTP headers.
   * @param context       OpenTelemetry's Context.
   * @return CompletableFuture for Response.
   */
private CompletableFuture<Response> doInvokeApi(String method,
                               String[] pathSegments,
                               Map<String, List<String>> urlParameters,
                               byte[] content, Map<String, String> headers,
                               Context context) {
    // 方法人口参数基本就是一个非常简化的HTTP请求的格式抽象

    // 取 UUID 为 requestId
    final String requestId = UUID.randomUUID().toString();
    RequestBody body;

    //组装 okhttp3 的 request
    String contentType = headers != null ? headers.get(Metadata.CONTENT_TYPE) : null;
    MediaType mediaType = contentType == null ? MEDIA_TYPE_APPLICATION_JSON : MediaType.get(contentType);
    if (content == null) {
      body = mediaType.equals(MEDIA_TYPE_APPLICATION_JSON)
          ? REQUEST_BODY_EMPTY_JSON
          : RequestBody.Companion.create(new byte[0], mediaType);
    } else {
      body = RequestBody.Companion.create(content, mediaType);
    }
    HttpUrl.Builder urlBuilder = new HttpUrl.Builder();
    urlBuilder.scheme(DEFAULT_HTTP_SCHEME)
        .host(this.hostname)
        .port(this.port);
    for (String pathSegment : pathSegments) {
      urlBuilder.addPathSegment(pathSegment);
    }
    Optional.ofNullable(urlParameters).orElse(Collections.emptyMap()).entrySet().stream()
        .forEach(urlParameter ->
            Optional.ofNullable(urlParameter.getValue()).orElse(Collections.emptyList()).stream()
              .forEach(urlParameterValue ->
                  urlBuilder.addQueryParameter(urlParameter.getKey(), urlParameterValue)));

    Request.Builder requestBuilder = new Request.Builder()
        .url(urlBuilder.build())
        .addHeader(HEADER_DAPR_REQUEST_ID, requestId);
    if (context != null) {
      context.stream()
          .filter(entry -> ALLOWED_CONTEXT_IN_HEADERS.contains(entry.getKey().toString().toLowerCase()))
          .forEach(entry -> requestBuilder.addHeader(entry.getKey().toString(), entry.getValue().toString()));
    }
    if (HttpMethods.GET.name().equals(method)) {
      requestBuilder.get();
    } else if (HttpMethods.DELETE.name().equals(method)) {
      requestBuilder.delete();
    } else {
      requestBuilder.method(method, body);
    }

    String daprApiToken = Properties.API_TOKEN.get();
    if (daprApiToken != null) {
      requestBuilder.addHeader(Headers.DAPR_API_TOKEN, daprApiToken);
    }

    if (headers != null) {
      Optional.ofNullable(headers.entrySet()).orElse(Collections.emptySet()).stream()
          .forEach(header -> {
            requestBuilder.addHeader(header.getKey(), header.getValue());
          });
    }
    // 完成 request 的组装，构建 request 对象
    Request request = requestBuilder.build();

    // 发出 okhttp3 的请求，然后返回 CompletableFuture
    CompletableFuture<Response> future = new CompletableFuture<>();
    this.httpClient.newCall(request).enqueue(new ResponseFutureCallback(future));
    return future;
  }
  ```

  在 http 请求组装过程中，注意 header 的处理：

  - request id： "X-DaprRequestId"，值为 UUID
  - dapr api token： "dapr-api-token"，值从系统变量 "dapr.api.token" 或者环境变量 "DAPR_API_TOKEN" 中获取
  - 发送请求时明确传递的header： 透传
  - OpenTelemetry 相关的值：会试图从传递进来的 OpenTelemetry context 中获取 "grpc-trace-bin", "traceparent", "tracestate" 这三个 header 并继续传递下去

