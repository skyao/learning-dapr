---
type: docs
title: "pod_patch.go的源码学习"
linkTitle: "pod_patch.go"
weight: 20050
date: 2021-05-11
description: >
  Dapr Injector 中的 pod_patch.go 的 代码
---



## 主流程

getPodPatchOperations() 是最重要的方法，injector 对 pod 的修改就在这里进行：

```go

func (i *injector) getPodPatchOperations(ar *v1.AdmissionReview,
	namespace, image, imagePullPolicy string, kubeClient *kubernetes.Clientset, daprClient scheme.Interface) ([]PatchOperation, error) {
    ......
  	return patchOps, nil
}
```

解析request，得到 pod 对象 （这里和前面重复了？）：

```go
req := ar.Request
var pod corev1.Pod
if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
   errors.Wrap(err, "could not unmarshal raw object")
   return nil, err
}
```

判断是否需要 injector 做处理：

```go
if !isResourceDaprEnabled(pod.Annotations) || podContainsSidecarContainer(&pod) {
   return nil, nil
}

// 判断是否启动了dapr，依据是是否设置 annotation "dapr.io/enabled" 为 true，默认为false
const daprEnabledKey                    = "dapr.io/enabled"
func isResourceDaprEnabled(annotations map[string]string) bool {
	return getBoolAnnotationOrDefault(annotations, daprEnabledKey, false)
}

// 判断是否包含了 dapr 的 sidecar container
const 	sidecarContainerName              = "daprd"
func podContainsSidecarContainer(pod *corev1.Pod) bool {
	for _, c := range pod.Spec.Containers {
    // 检测方式是循环pod中的所有container，检查是否有container的名字为 "daprd"
		if c.Name == sidecarContainerName {
			return true
		}
	}
	return false
}
```

创建 daprd sidecar container：

```go
sidecarContainer, err := getSidecarContainer(pod.Annotations, id, image, imagePullPolicy, req.Namespace, apiSrvAddress, placementAddress, tokenMount, trustAnchors, certChain, certKey, sentryAddress, mtlsEnabled, identity)
```

getSidecarContainer（）的细节后面看，先走完主流程。

```go
patchOps := []PatchOperation{}
envPatchOps := []PatchOperation{}
var path string
var value interface{}
if len(pod.Spec.Containers) == 0 {
   // 如果pod的container数量为0（什么情况下会有这种没有container的pod？）
   path = containersPath
   value = []corev1.Container{*sidecarContainer}
} else {
   // 将 daprd 的sidecar 加入
   envPatchOps = addDaprEnvVarsToContainers(pod.Spec.Containers)
   // TODO：path 的设值有什么规范或者要求？
   path = "/spec/containers/-"
   value = sidecarContainer
}

	patchOps = append(
		patchOps,
		PatchOperation{
			Op:    "add",
			Path:  path,
			Value: value,
		},
	)
	patchOps = append(patchOps, envPatchOps...)
```



### addDaprEnvVarsToContainers

```go
// This function add Dapr environment variables to all the containers in any Dapr enabled pod.
// The containers can be injected or user defined.
func addDaprEnvVarsToContainers(containers []corev1.Container) []PatchOperation {
   portEnv := []corev1.EnvVar{
      {
         Name:  userContainerDaprHTTPPortName,
         Value: strconv.Itoa(sidecarHTTPPort),
      },
      {
         Name:  userContainerDaprGRPCPortName,
         Value: strconv.Itoa(sidecarAPIGRPCPort),
      },
   }
   envPatchOps := make([]PatchOperation, 0, len(containers))
   for i, container := range containers {
      path := fmt.Sprintf("%s/%d/env", containersPath, i)
      patchOps := getEnvPatchOperations(container.Env, portEnv, path)
      envPatchOps = append(envPatchOps, patchOps...)
   }
   return envPatchOps
}
```



## 分支流程：mTLS的处理

```go
mtlsEnabled := mTLSEnabled(daprClient)
if mtlsEnabled {
   trustAnchors, certChain, certKey = getTrustAnchorsAndCertChain(kubeClient, namespace)
   identity = fmt.Sprintf("%s:%s", req.Namespace, pod.Spec.ServiceAccountName)
}
```



```go
func mTLSEnabled(daprClient scheme.Interface) bool {
   resp, err := daprClient.ConfigurationV1alpha1().Configurations(meta_v1.NamespaceAll).List(meta_v1.ListOptions{})
   if err != nil {
      log.Errorf("Failed to load dapr configuration from k8s, use default value %t for mTLSEnabled: %s", defaultMtlsEnabled, err)
      return defaultMtlsEnabled
   }

   for _, c := range resp.Items {
      if c.GetName() == defaultConfig {
         return c.Spec.MTLSSpec.Enabled
      }
   }
   log.Infof("Dapr system configuration (%s) is not found, use default value %t for mTLSEnabled", defaultConfig, defaultMtlsEnabled)
   return defaultMtlsEnabled
}
```

## 分支处理：serviceaccount

