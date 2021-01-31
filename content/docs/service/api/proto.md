---
title: "服务调用API的Proto定义"
linkTitle: "Proto定义"
weight: 522
date: 2021-01-29
description: >
  Dapr服务调用API的Proto定义
---

### InvokeService的定义

Service Invoken API 定义在 proto文件  [dapr/proto/runtime/v1/dapr.proto](https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/dapr/proto/runtime/v1/dapr.proto) 中：

```protobuf
service Dapr {
  // Invokes a method on a remote Dapr app.
  rpc InvokeService(InvokeServiceRequest) returns (common.v1.InvokeResponse) {}
  ...
}
```

InvokeServiceRequest 包含一个被调用服务的ID，和通用的 InvokeRequest：

```protobuf
// InvokeServiceRequest represents the request message for Service invocation.
message InvokeServiceRequest {
  // Required. Callee's app id.
  string id = 1;

  // Required. message which will be delivered to callee.
  common.v1.InvokeRequest message = 3;
}
```

### AppCallback的定义

AppCallback API 定义在 proto文件  [dapr/proto/runtime/v1/appcallback.proto](https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/dapr/proto/runtime/v1/appcallback.proto) 中：

```protobuf
service AppCallback {
  // Invokes service method with InvokeRequest.
  rpc OnInvoke (common.v1.InvokeRequest) returns (common.v1.InvokeResponse) {}
  ...
}
```



### Invoke的通用定义

Invoke的通用定义在 proto文件  [dapr/proto/common/v1/common.proto](https://github.com/dapr/dapr/blob/de49fe260c8f7c53e146e27150faad8c0880fe90/dapr/proto/common/v1/common.proto) 中。

InvokeRequest是用来携带数据调用方法的消息，这个消息在Dapr gRPC服务的InvokeService和AppCallback gRPC服务的OnInvoke中使用：

```protobuf
// InvokeRequest is the message to invoke a method with the data.
// This message is used in InvokeService of Dapr gRPC Service and OnInvoke
// of AppCallback gRPC service.
message InvokeRequest {
  // Required. method is a method name which will be invoked by caller.
  string method = 1;

  // Required. Bytes value or Protobuf message which caller sent.
  // Dapr treats Any.value as bytes type if Any.type_url is unset.
  google.protobuf.Any data = 2;

  // The type of data content.
  //
  // This field is required if data delivers http request body
  // Otherwise, this is optional.
  string content_type = 3;

  // HTTP specific fields if request conveys http-compatible request.
  //
  // This field is required for http-compatible request. Otherwise,
  // this field is optional.
  HTTPExtension http_extension = 4;
}
```

InvokeResponse是包括应用程序回调的数据和内容类型的响应消息，该消息在Dapr gRPC服务的InvokeService方法和AppCallback gRPC服务的OnInvoke方法中使用：

```protobuf
// InvokeResponse is the response message inclduing data and its content type
// from app callback.
// This message is used in InvokeService of Dapr gRPC Service and OnInvoke
// of AppCallback gRPC service.
message InvokeResponse {
  // Required. The content body of InvokeService response.
  google.protobuf.Any data = 1;

  // Required. The type of data content.
  string content_type = 2;
}
```

### 相关的消息定义

HTTPExtension 消息的定义：

```protobuf
// 当Dapr运行时传递HTTP内容时，HTTPExtension包括HTTP verb和querystring。
// 
// For example, when callers calls http invoke api
// POST http://localhost:3500/v1.0/invoke/<app_id>/method/<method>?query1=value1&query2=value2
// 
// Dapr runtime will parse POST as a verb and extract querystring to quersytring map.
message HTTPExtension {
  // Type of HTTP 1.1 Methods
  // RFC 7231: https://tools.ietf.org/html/rfc7231#page-24
  enum Verb {
    NONE = 0;
    GET = 1;
    HEAD = 2;
    POST = 3;
    PUT = 4;
    DELETE = 5;
    CONNECT = 6;
    OPTIONS = 7;
    TRACE = 8;
  }

  // Required. HTTP verb.
  Verb verb = 1;

  // querystring includes HTTP querystring.
  map<string, string> querystring = 2;
}
```

Any消息的定义：

```protobuf
message Any {
  string type_url = 1;

  // Must be a valid serialized protocol buffer of the above specified type.
  bytes value = 2;
}
```







