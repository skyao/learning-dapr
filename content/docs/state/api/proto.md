---
title: "状态管理API的Proto定义"
linkTitle: "Proto定义"
weight: 822
date: 2021-01-31
description: >
  Dapr状态管理API的Proto定义
---




### State API的定义

State Management API 定义在 proto文件  [dapr/proto/runtime/v1/dapr.proto](https://github.com/dapr/dapr/blob/11741c6cd697e08b2e776943e61bb2e3388c85a8/dapr/proto/runtime/v1/dapr.proto) 中：

```protobuf
service Dapr {
  // Gets the state for a specific key.
  rpc GetState(GetStateRequest) returns (GetStateResponse) {}

  // Gets a bulk of state items for a list of keys
  rpc GetBulkState(GetBulkStateRequest) returns (GetBulkStateResponse) {}

  // Saves the state for a specific key.
  rpc SaveState(SaveStateRequest) returns (google.protobuf.Empty) {}

  // Deletes the state for a specific key.
  rpc DeleteState(DeleteStateRequest) returns (google.protobuf.Empty) {}
  
  // Executes transactions for a specified store
  rpc ExecuteStateTransaction(ExecuteStateTransactionRequest) returns (google.protobuf.Empty) {}
  ...
}
```

另外的common.proto中定义了和state相关的消息和枚举：

```protobuf
// StateOptions configures concurrency and consistency for state operations
message StateOptions {
  // Enum describing the supported concurrency for state.
  enum StateConcurrency {
    CONCURRENCY_UNSPECIFIED = 0;
    CONCURRENCY_FIRST_WRITE = 1;
    CONCURRENCY_LAST_WRITE = 2;
  }

  // Enum describing the supported consistency for state.
  enum StateConsistency {
    CONSISTENCY_UNSPECIFIED = 0;
    CONSISTENCY_EVENTUAL = 1;
    CONSISTENCY_STRONG = 2;
  }

  StateConcurrency concurrency = 1;
  StateConsistency consistency = 2;
}
```

### get state

GetStateRequest 包含store_name/key，还有并发要求和请求级别的metadata：

```protobuf
// GetStateRequest is the message to get key-value states from specific state store.
message GetStateRequest {
  // The name of state store.
  string store_name = 1;

  // The key of the desired state
  string key = 2;

  // The read consistency of the state store.
  common.v1.StateOptions.StateConsistency consistency = 3;

  // The metadata which will be sent to state store components.
  map<string,string> metadata = 4;
}
```

GetStateResponse 包含byte[] 形式的 state 数据 data，和特殊表示数据特定版本的etag：

```protobuf
// GetStateResponse is the response conveying the state value and etag.
message GetStateResponse {
  // The byte array data
  bytes data = 1;

  // The entity tag which represents the specific version of data.
  // ETag format is defined by the corresponding data store.
  string etag = 2;
}
```

### Get Bulk State

GetBulkStateRequest 是批量接口，一次性获取多个key的数据：

```protobuf
// GetBulkStateRequest is the message to get a list of key-value states from specific state store.
message GetBulkStateRequest {
  // The name of state store.
  string store_name = 1;

  // The keys to get.
  repeated string keys = 2;

  // The number of parallel operations executed on the state store for a get operation.
  // 在状态存储上用于get操作的并行操作执行的数量：也就是并发数，同时执行的请求数量
  int32 parallelism = 3;

  // The metadata which will be sent to state store components.
  // 请求级别，意味着所有的key都是使用同样的metadata
  map<string,string> metadata = 4;
}
```

GetBulkStateResponse：

```protobuf
// GetBulkStateResponse is the response conveying the list of state values.
message GetBulkStateResponse {
  // The list of items containing the keys to get values for.
  // 为啥不用map？
  repeated BulkStateItem items = 1;
}

// BulkStateItem is the response item for a bulk get operation.
// Return values include the item key, data and etag.
message BulkStateItem {
  // state item key
  string key = 1;

  // The byte array data
  bytes data = 2;

  // The entity tag which represents the specific version of data.
  // ETag format is defined by the corresponding data store.
  string etag = 3;

  // The error that was returned from the state store in case of a failed get operation.
  // 这里考虑了出错的可能，有机会给出错误信息
  // 但是，单个get state 操作怎么没有定义错误信息？
  // 只能在http/grpc协议层上报错？TODO：看看代码实现
  string error = 4;
}
```

### Save State

SaveStateRequest 支持多个状态的保存：

```protobuf
// SaveStateRequest is the message to save multiple states into state store.
message SaveStateRequest {
  // The name of state store.
  string store_name = 1;

  // The array of the state key values.
  repeated common.v1.StateItem states = 2;
}

// StateItem represents state key, value, and additional options to save state.
message StateItem {
  // Required. The state key
  string key = 1;

  // Required. The state data for key
  bytes value = 2;

  // The entity tag which represents the specific version of data.
  // The exact ETag format is defined by the corresponding data store.
  string etag = 3;

  // The metadata which will be passed to state store component.
  map<string,string> metadata = 4;

  // Options for concurrency and consistency to save the state.
  StateOptions options = 5;
}
```

response为 google.protobuf.Empty。



### Delete State

DeleteStateRequest：

```protobuf
// DeleteStateRequest is the message to delete key-value states in the specific state store.
message DeleteStateRequest {
  // The name of state store.
  string store_name = 1;

  // The key of the desired state
  string key = 2;

  // The entity tag which represents the specific version of data.
  // The exact ETag format is defined by the corresponding data store.
  string etag = 3;

  // State operation options which includes concurrency/
  // consistency/retry_policy.
  common.v1.StateOptions options = 4;

  // The metadata which will be sent to state store components.
  map<string,string> metadata = 5;
}
```

response为 google.protobuf.Empty。

### Execute State Transaction

ExecuteStateTransactionRequest

```protobuf
// ExecuteStateTransactionRequest is the message to execute multiple operations on a specified store.
message ExecuteStateTransactionRequest {
  // Required. name of state store.
  string storeName = 1;

  // Required. transactional operation list.
  repeated TransactionalStateOperation operations = 2;

  // The metadata used for transactional operations.
  map<string,string> metadata = 3;
}

// TransactionalStateOperation is the message to execute a specified operation with a key-value pair.
message TransactionalStateOperation {
  // The type of operation to be executed
  // 具体有哪些操作？
  string operationType = 1;

  // State values to be operated on 
  common.v1.StateItem request = 2;
}
```

response为 google.protobuf.Empty。