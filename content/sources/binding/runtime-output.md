---
type: docs
title: "资源绑定的output处理源码分析"
linkTitle: "output处理"
weight: 734
date: 2021-01-31
description: >
  Dapr资源绑定的output处理源码分析
---


`pkc/grpc/api.go` 中的 InvokeBinding 方法：

```go
func (a *api) InvokeBinding(ctx context.Context, in *runtimev1pb.InvokeBindingRequest) (*runtimev1pb.InvokeBindingResponse, error) {
	req := &bindings.InvokeRequest{
		Metadata:  in.Metadata,
		Operation: bindings.OperationKind(in.Operation),
	}
	if in.Data != nil {
		req.Data = in.Data
	}

	r := &runtimev1pb.InvokeBindingResponse{}
  // 关键实现在这里
	resp, err := a.sendToOutputBindingFn(in.Name, req)
	if err != nil {
		err = fmt.Errorf("ERR_INVOKE_OUTPUT_BINDING: %s", err)
		apiServerLogger.Debug(err)
		return r, err
	}

	if resp != nil {
		r.Data = resp.Data
		r.Metadata = resp.Metadata
	}
	return r, nil
}
```

sendToOutputBindingFn 方法的初始化在这里：

```go
func (a *DaprRuntime) getGRPCAPI() grpc.API {
	return grpc.NewAPI(a.runtimeConfig.ID, a.appChannel, a.stateStores, a.secretStores, a.getPublishAdapter(), a.directMessaging, a.actor, a.sendToOutputBinding, a.globalConfig.Spec.TracingSpec)
}
```

sendToOutputBinding 方法的实现在 `pkg/runtime/runtime.go`:

```
func (a *DaprRuntime) sendToOutputBinding(name string, req *bindings.InvokeRequest) (*bindings.InvokeResponse, error) {
   if req.Operation == "" {
      return nil, errors.New("operation field is missing from request")
   }

   // 根据 name 找已经注册好的 binding
   if binding, ok := a.outputBindings[name]; ok {
      ops := binding.Operations()
      for _, o := range ops {
      	 // 找到改 binding 下支持的 operation
         if o == req.Operation {
         		// 关键代码，需要转到具体的实现了
            return binding.Invoke(req)
         }
      }
      supported := make([]string, len(ops))
      for _, o := range ops {
         supported = append(supported, string(o))
      }
      return nil, errors.Errorf("binding %s does not support operation %s. supported operations:%s", name, req.Operation, strings.Join(supported, " "))
   }
   return nil, errors.Errorf("couldn't find output binding %s", name)
}
```



