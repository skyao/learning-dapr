---
title: "状态管理的runtime处理源码分析"
linkTitle: "runtime处理"
weight: 833
date: 2021-01-31
description: >
  Dapr状态管理的runtime处理源码分析
---


runtime 处理 state 请求的代码在 `pkg/grpc/api.go` 中。

### get state

```go
func (a *api) GetState(ctx context.Context, in *runtimev1pb.GetStateRequest) (*runtimev1pb.GetStateResponse, error) {
  // 找 store name 对应的 state store
  // 所以请求里面的 store name，必须对应 yaml 文件里面的 name
	store, err := a.getStateStore(in.StoreName)
	if err != nil {
		apiServerLogger.Debug(err)
		return &runtimev1pb.GetStateResponse{}, err
	}

	req := state.GetRequest{
		Key:      a.getModifiedStateKey(in.Key),
		Metadata: in.Metadata,
		Options: state.GetStateOption{
			Consistency: stateConsistencyToString(in.Consistency),
		},
	}

  // 执行查询
  // 里面实际上会先执行 HGETALL 命令，失败后再执行 GET 命令
	getResponse, err := store.Get(&req)
	if err != nil {
		err = fmt.Errorf("ERR_STATE_GET: %s", err)
		apiServerLogger.Debug(err)
		return &runtimev1pb.GetStateResponse{}, err
	}

	response := &runtimev1pb.GetStateResponse{}
	if getResponse != nil {
		response.Etag = getResponse.ETag
		response.Data = getResponse.Data
	}
	return response, nil
}
```

### get bulk state

get bulk 方法的实现是有 runtime 封装 get 方法而成，底层 state store 只需要实现单个查询的 get 即可。

```go
func (a *api) GetBulkState(ctx context.Context, in *runtimev1pb.GetBulkStateRequest) (*runtimev1pb.GetBulkStateResponse, error) {
   store, err := a.getStateStore(in.StoreName)
   if err != nil {
      apiServerLogger.Debug(err)
      return &runtimev1pb.GetBulkStateResponse{}, err
   }

   resp := &runtimev1pb.GetBulkStateResponse{}
   // 如果 Parallelism <= 0，则取默认值100
   limiter := concurrency.NewLimiter(int(in.Parallelism))

   for _, k := range in.Keys {
      fn := func(param interface{}) {
         req := state.GetRequest{
            Key:      a.getModifiedStateKey(param.(string)),
            Metadata: in.Metadata,
         }

         r, err := store.Get(&req)
         item := &runtimev1pb.BulkStateItem{
            Key: param.(string),
         }
         if err != nil {
            item.Error = err.Error()
         } else if r != nil {
            item.Data = r.Data
            item.Etag = r.ETag
         }
         resp.Items = append(resp.Items, item)
      }

      limiter.Execute(fn, k)
   }
   limiter.Wait()

   return resp, nil
}
```

### save state

```go
func (a *api) SaveState(ctx context.Context, in *runtimev1pb.SaveStateRequest) (*empty.Empty, error) {
   store, err := a.getStateStore(in.StoreName)
   if err != nil {
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }

   reqs := []state.SetRequest{}
   for _, s := range in.States {
      req := state.SetRequest{
         Key:      a.getModifiedStateKey(s.Key),
         Metadata: s.Metadata,
         Value:    s.Value,
         ETag:     s.Etag,
      }
      if s.Options != nil {
         req.Options = state.SetStateOption{
            Consistency: stateConsistencyToString(s.Options.Consistency),
            Concurrency: stateConcurrencyToString(s.Options.Concurrency),
         }
      }
      reqs = append(reqs, req)
   }

   // 调用 store 的 BulkSet 方法
   // 事实上store的Set方法根本没有被 runtime 调用？？？
   err = store.BulkSet(reqs)
   if err != nil {
      err = fmt.Errorf("ERR_STATE_SAVE: %s", err)
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }
   return &empty.Empty{}, nil
}
```

### delete state

```go
func (a *api) DeleteState(ctx context.Context, in *runtimev1pb.DeleteStateRequest) (*empty.Empty, error) {
   store, err := a.getStateStore(in.StoreName)
   if err != nil {
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }

   req := state.DeleteRequest{
      Key:      a.getModifiedStateKey(in.Key),
      Metadata: in.Metadata,
      ETag:     in.Etag,
   }
   if in.Options != nil {
      req.Options = state.DeleteStateOption{
         Concurrency: stateConcurrencyToString(in.Options.Concurrency),
         Consistency: stateConsistencyToString(in.Options.Consistency),
      }
   }

   // 调用 store 的delete方法
   // store 的 BulkDelete 方法没有调用
   // runtime 也没有对外暴露 BulkDelete 方法
   err = store.Delete(&req)
   if err != nil {
      err = fmt.Errorf("ERR_STATE_DELETE: failed deleting state with key %s: %s", in.Key, err)
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }
   return &empty.Empty{}, nil
}
```

### Execute State Transaction

如果要支持事务，则要求实现 TransactionalStore 接口：

```
type TransactionalStore interface {
   // Init方法是和普通store接口一致的
   Init(metadata Metadata) error
   // 增加的是 Multi 方法
   Multi(request *TransactionalStateRequest) error
}
```

runtime 的 ExecuteStateTransaction 方法的实现：

```go
func (a *api) ExecuteStateTransaction(ctx context.Context, in *runtimev1pb.ExecuteStateTransactionRequest) (*empty.Empty, error) {
   if a.stateStores == nil || len(a.stateStores) == 0 {
      err := errors.New("ERR_STATE_STORE_NOT_CONFIGURED")
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }

   storeName := in.StoreName

   if a.stateStores[storeName] == nil {
      err := errors.New("ERR_STATE_STORE_NOT_FOUND")
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }

   // 检测是否是 TransactionalStore
   transactionalStore, ok := a.stateStores[storeName].(state.TransactionalStore)
   if !ok {
      err := errors.New("ERR_STATE_STORE_NOT_SUPPORTED")
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }

   // 构造请求
   operations := []state.TransactionalStateOperation{}
   for _, inputReq := range in.Operations {
      var operation state.TransactionalStateOperation
      var req = inputReq.Request
      switch state.OperationType(inputReq.OperationType) {
      case state.Upsert:
         setReq := state.SetRequest{
            Key: a.getModifiedStateKey(req.Key),
            // Limitation:
            // components that cannot handle byte array need to deserialize/serialize in
            // component sepcific way in components-contrib repo.
            Value:    req.Value,
            Metadata: req.Metadata,
            ETag:     req.Etag,
         }

         if req.Options != nil {
            setReq.Options = state.SetStateOption{
               Concurrency: stateConcurrencyToString(req.Options.Concurrency),
               Consistency: stateConsistencyToString(req.Options.Consistency),
            }
         }

         operation = state.TransactionalStateOperation{
            Operation: state.Upsert,
            Request:   setReq,
         }

      case state.Delete:
         delReq := state.DeleteRequest{
            Key:      a.getModifiedStateKey(req.Key),
            Metadata: req.Metadata,
            ETag:     req.Etag,
         }

         if req.Options != nil {
            delReq.Options = state.DeleteStateOption{
               Concurrency: stateConcurrencyToString(req.Options.Concurrency),
               Consistency: stateConsistencyToString(req.Options.Consistency),
            }
         }

         operation = state.TransactionalStateOperation{
            Operation: state.Delete,
            Request:   delReq,
         }

      default:
         err := fmt.Errorf("ERR_OPERATION_NOT_SUPPORTED: operation type %s not supported", inputReq.OperationType)
         apiServerLogger.Debug(err)
         return &empty.Empty{}, err
      }

      operations = append(operations, operation)
   }
 
   // 调用 state store 的 Multi 方法执行有事务性的多个操作
   err := transactionalStore.Multi(&state.TransactionalStateRequest{
      Operations: operations,
      Metadata:   in.Metadata,
   })

   if err != nil {
      err = fmt.Errorf("ERR_STATE_TRANSACTION: %s", err)
      apiServerLogger.Debug(err)
      return &empty.Empty{}, err
   }
   return &empty.Empty{}, nil
}
```