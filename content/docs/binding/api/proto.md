---
title: "资源绑定API的Proto定义"
linkTitle: "Proto定义"
weight: 722
date: 2021-01-31
description: >
  Dapr的资源绑定API的Proto定义
---



### Output Binding的定义

Output Binding API 定义在 proto文件  [dapr/proto/runtime/v1/dapr.proto](https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/dapr/proto/runtime/v1/dapr.proto) 中：

```protobuf
service Dapr {
  // Invokes binding data to specific output bindings
  rpc InvokeBinding(InvokeBindingRequest) returns (InvokeBindingResponse) {}
  ...
}
```

InvokeBindingRequest 包含一个被绑定资源的name，数据data，元数据metadata和绑定资源的操作类型operaion：

```protobuf
// InvokeBindingRequest is the message to send data to output bindings
message InvokeBindingRequest {
  // The name of the output binding to invoke.
  string name = 1;

  // The data which will be sent to output binding.
  bytes data = 2;

  // The metadata passing to output binding components
  // 
  // Common metadata property:
  // - ttlInSeconds : the time to live in seconds for the message. 
  // If set in the binding definition will cause all messages to 
  // have a default time to live. The message ttl overrides any value
  // in the binding definition.
  map<string,string> metadata = 3;

  // The name of the operation type for the binding to invoke
  string operation = 4;
}
```

InvokeBindingResponse 的定义：

```protobuf
// InvokeBindingResponse is the message returned from an output binding invocation
message InvokeBindingResponse {
  // The data which will be sent to output binding.
  bytes data = 1;

  // The metadata returned from an external system
  map<string,string> metadata = 2;
}
```

### Input Binding的定义

Input Binding API 定义在 proto文件  [dapr/proto/runtime/v1/appcallback.proto](https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/dapr/proto/runtime/v1/appcallback.proto) 中：

```protobuf
service AppCallback {
  // Lists all input bindings subscribed by this app.
  rpc ListInputBindings(google.protobuf.Empty) returns (ListInputBindingsResponse) {}
  
  // Listens events from the input bindings
  //
  // User application can save the states or send the events to the output
  // bindings optionally by returning BindingEventResponse.
  rpc OnBindingEvent(BindingEventRequest) returns (BindingEventResponse) {}
  ...
}
```

ListInputBindingsResponse 的定义：

```protobuf
// ListInputBindingsResponse is the message including the list of input bindings.
message ListInputBindingsResponse {
  // The list of input bindings.
  repeated string bindings = 1;
}
```

BindingEventRequest的定义：

```protobuf
// BindingEventRequest represents input bindings event.
message BindingEventRequest {
  // Requried. The name of the input binding component.
  string name = 1;

  // Required. The payload that the input bindings sent
  bytes data = 2;

  // The metadata set by the input binging components.
  map<string,string> metadata = 3;
}
```

BindingEventResponse 的定义：

```protobuf
// BindingEventResponse includes operations to save state or
// send data to output bindings optionally.
message BindingEventResponse {
  // The name of state store where states are saved.
  string store_name = 1;

  // The state key values which will be stored in store_name.
  repeated common.v1.StateItem states = 2;

  // BindingEventConcurrency is the kind of concurrency 
  enum BindingEventConcurrency {
    // SEQUENTIAL sends data to output bindings specified in "to" sequentially.
    SEQUENTIAL = 0;
    // PARALLEL sends data to output bindings specified in "to" in parallel.
    PARALLEL = 1;
  }

  // The list of output bindings.
  repeated string to = 3;

  // The content which will be sent to "to" output bindings.
  bytes data = 4;

  // The concurrency of output bindings to send data to
  // "to" output bindings list. The default is SEQUENTIAL.
  BindingEventConcurrency concurrency = 5;
}
```

