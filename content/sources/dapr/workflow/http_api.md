---
title: "workflow HTTP API"
linkTitle: "HTTP API实现"
weight: 20
date: 2021-01-31
description: >
  Dapr workflow的HTTP API实现
---

`pkg/http/api.go`



### 构建workflow的endpoint

```go
const (
    workflowComponent        = "workflowComponent"
	workflowName             = "workflowName"
)

func NewAPI(opts APIOpts) API {
	api := &api{
        ......
	api.endpoints = append(api.endpoints, api.constructWorkflowEndpoints()...)
	return api
}
    
```

constructWorkflowEndpoints() 方法的实现在 `pkg/http/api_workflow.go` 中：

```go
func (a *api) constructWorkflowEndpoints() []Endpoint {
	return []Endpoint{
		{
			Methods: []string{http.MethodGet},
			Route:   "workflows/{workflowComponent}/{instanceID}",
			Version: apiVersionV1alpha1,
			Handler: a.onGetWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{instanceID}/raiseEvent/{eventName}",
			Version: apiVersionV1alpha1,
			Handler: a.onRaiseEventWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{workflowName}/start",
			Version: apiVersionV1alpha1,
			Handler: a.onStartWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{instanceID}/pause",
			Version: apiVersionV1alpha1,
			Handler: a.onPauseWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{instanceID}/resume",
			Version: apiVersionV1alpha1,
			Handler: a.onResumeWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{instanceID}/terminate",
			Version: apiVersionV1alpha1,
			Handler: a.onTerminateWorkflowHandler(),
		},
		{
			Methods: []string{http.MethodPost},
			Route:   "workflows/{workflowComponent}/{instanceID}/purge",
			Version: apiVersionV1alpha1,
			Handler: a.onPurgeWorkflowHandler(),
		},
	}
}
```

## handler 实现

`pkg/http/api_workflow.go`

### onStartWorkflowHandler()

```go
// Route:   "workflows/{workflowComponent}/{workflowName}/start?instanceID={instanceID}",
// Workflow Component: Component specified in yaml
// Workflow Name: Name of the workflow to run
// Instance ID: Identifier of the specific run
func (a *api) onStartWorkflowHandler() http.HandlerFunc {
	return UniversalHTTPHandler(
		a.universal.StartWorkflowAlpha1,
        // UniversalHTTPHandlerOpts 是范型结构体
		UniversalHTTPHandlerOpts[*runtimev1pb.StartWorkflowRequest, *runtimev1pb.StartWorkflowResponse]{
			// We pass the input body manually rather than parsing it using protojson
			SkipInputBody: true,
			InModifier: func(r *http.Request, in *runtimev1pb.StartWorkflowRequest) (*runtimev1pb.StartWorkflowRequest, error) {
				in.WorkflowName = chi.URLParam(r, workflowName)
				in.WorkflowComponent = chi.URLParam(r, workflowComponent)

                // instance id 是可选的，如果没有指定则生成一个随机的
				// The instance ID is optional. If not specified, we generate a random one.
				in.InstanceId = r.URL.Query().Get(instanceID)
				if in.InstanceId == "" {
					randomID, err := uuid.NewRandom()
					if err != nil {
						return nil, err
					}
					in.InstanceId = randomID.String()
				}

                // HTTP request body 直接用来做 workflow 的 Input
				// We accept the HTTP request body as the input to the workflow
				// without making any assumptions about its format.
				var err error
				in.Input, err = io.ReadAll(r.Body)
				if err != nil {
					return nil, messages.ErrBodyRead.WithFormat(err)
				}
				return in, nil
			},
			SuccessStatusCode: http.StatusAccepted,
		})
}
```

### onGetWorkflowHandler()

```go
// Route: POST "workflows/{workflowComponent}/{instanceID}"
func (a *api) onGetWorkflowHandler() http.HandlerFunc {
	return UniversalHTTPHandler(
		a.universal.GetWorkflowAlpha1,
		UniversalHTTPHandlerOpts[*runtimev1pb.GetWorkflowRequest, *runtimev1pb.GetWorkflowResponse]{
			InModifier: workflowInModifier[*runtimev1pb.GetWorkflowRequest],
		})
}
```

workflowInModifier() 方法是通用方法，读取 WorkflowComponent 和 InstanceId 两个参数：

```go
// Shared InModifier method for all universal handlers for workflows that adds the "WorkflowComponent" and "InstanceId" properties
func workflowInModifier[T runtimev1pb.WorkflowRequests](r *http.Request, in T) (T, error) {
	in.SetWorkflowComponent(chi.URLParam(r, workflowComponent))
	in.SetInstanceId(chi.URLParam(r, instanceID))
	return in, nil
}
```

