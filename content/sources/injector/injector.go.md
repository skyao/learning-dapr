---
type: docs
title: "injector.go的源码学习"
linkTitle: "injector.go"
weight: 20050
date: 2021-05-11
description: >
  Dapr Injector 中的 injector.go 的 代码
---

## 主流程代码

### 接口和结构体定义和创建

Injector 是Dapr运行时 sidecar 注入组件的接口。

```go
// Injector is the interface for the Dapr runtime sidecar injection component
type Injector interface {
   Run(ctx context.Context)
}
```

injector 结构体定义：

```go
type injector struct {
   config       Config
   deserializer runtime.Decoder
   server       *http.Server
   kubeClient   *kubernetes.Clientset
   daprClient   scheme.Interface
   authUIDs     []string
}
```

创建新的 injector 结构体（这个方法在injecot的main方法中被调用）：

```go
// NewInjector returns a new Injector instance with the given config
func NewInjector(authUIDs []string, config Config, daprClient scheme.Interface, kubeClient *kubernetes.Clientset) Injector {
   mux := http.NewServeMux()

   i := &injector{
      config: config,
      deserializer: serializer.NewCodecFactory(
         runtime.NewScheme(),
      ).UniversalDeserializer(),
      // 启动http server
      server: &http.Server{
         Addr:    fmt.Sprintf(":%d", port),
         Handler: mux,
      },
      kubeClient: kubeClient,
      daprClient: daprClient,
      authUIDs:   authUIDs,
   }

   // 给 k8s 调用的 mutate 端点
   mux.HandleFunc("/mutate", i.handleRequest)
   return i
}
```

### Run()方法

最核心的run方法，

```go
func (i *injector) Run(ctx context.Context) {
   doneCh := make(chan struct{})

   // 启动go routing，监听 ctx 和 doneCh 的信号
   go func() {
      select {
      case <-ctx.Done():
         log.Info("Sidecar injector is shutting down")
         shutdownCtx, cancel := context.WithTimeout(
            context.Background(),
            time.Second*5,
         )
         defer cancel()
         i.server.Shutdown(shutdownCtx) // nolint: errcheck
      case <-doneCh:
      }
   }()

   // 打印启动时的日志，这行日志可以通过 
   log.Infof("Sidecar injector is listening on %s, patching Dapr-enabled pods", i.server.Addr)
   // TODO：这里有时会报错，证书有问题，导致injector无法正常工作，后面再来检查
   err := i.server.ListenAndServeTLS(i.config.TLSCertFile, i.config.TLSKeyFile)
   if err != http.ErrServerClosed {
      log.Errorf("Sidecar injector error: %s", err)
   }
   close(doneCh)
}
```

可以对比通过 `k logs dapr-sidecar-injector-86b8dc4dcd-bkbgw -n dapr-system` 命令查看到的injecot 日志内容：

```json
{"instance":"dapr-sidecar-injector-86b8dc4dcd-bkbgw","level":"info","msg":"log level set to: info","scope":"dapr.injector","time":"2021-05-11T01:13:20.1904136Z","type":"log","ver":"unknown"}
{"instance":"dapr-sidecar-injector-86b8dc4dcd-bkbgw","level":"info","msg":"metrics server started on :9090/","scope":"dapr.metrics","time":"2021-05-11T01:13:20.1907347Z","type":"log","ver":"unknown"}
{"instance":"dapr-sidecar-injector-86b8dc4dcd-bkbgw","level":"info","msg":"starting Dapr Sidecar Injector -- version edge -- commit v1.0.0-rc.4-163-g9a4210a-dirty","scope":"dapr.injector","time":"2021-05-11T01:13:20.191669Z","type":"log","ver":"unknown"}
{"instance":"dapr-sidecar-injector-86b8dc4dcd-bkbgw","level":"info","msg":"Healthz server is listening on :8080","scope":"dapr.injector","time":"2021-05-11T01:13:20.1928941Z","type":"log","ver":"unknown"}

{"instance":"dapr-sidecar-injector-86b8dc4dcd-bkbgw","level":"info","msg":"Sidecar injector is listening on :4000, patching Dapr-enabled pods","scope":"dapr.injector","time":"2021-05-11T01:13:20.208587Z","type":"log","ver":"unknown"}
```

### handleRequest方法

handleRequest方法用来处理来自 k8s api server的 mutate 调用：

```go
mux.HandleFunc("/mutate", i.handleRequest)

func (i *injector) handleRequest(w http.ResponseWriter, r *http.Request) {
  ......
}
```

代码比较长，忽略部分细节代码。

读取请求的body，验证长度和content-type：

```go
defer r.Body.Close()

var body []byte
if r.Body != nil {
   if data, err := ioutil.ReadAll(r.Body); err == nil {
      body = data
   }
}
if len(body) == 0 {
   log.Error("empty body")
   http.Error(w, "empty body", http.StatusBadRequest)
   return
}

contentType := r.Header.Get("Content-Type")
if contentType != "application/json" {
  log.Errorf("Content-Type=%s, expect application/json", contentType)
  http.Error(
    w,
    "invalid Content-Type, expect `application/json`",
    http.StatusUnsupportedMediaType,
  )

  return
}
```

反序列化body，并做一些基本的验证：

```go
ar := v1.AdmissionReview{}
_, gvk, err := i.deserializer.Decode(body, nil, &ar)
if err != nil {
   log.Errorf("Can't decode body: %v", err)
} else {
   if !utils.StringSliceContains(ar.Request.UserInfo.UID, i.authUIDs) {
      err = errors.Wrapf(err, "unauthorized request")
      log.Error(err)
   } else if ar.Request.Kind.Kind != "Pod" {
      err = errors.Wrapf(err, "invalid kind for review: %s", ar.Kind)
      log.Error(err)
   } else {
      patchOps, err = i.getPodPatchOperations(&ar, i.config.Namespace, i.config.SidecarImage, i.config.SidecarImagePullPolicy, i.kubeClient, i.daprClient)
   }
}
```

getPodPatchOperations 是核心代码，后面细看。

统一处理前面可能产生的错误，以及 getPodPatchOperations() 的处理结果：

```go
diagAppID := getAppIDFromRequest(ar.Request)

if err != nil {
   admissionResponse = toAdmissionResponse(err)
   log.Errorf("Sidecar injector failed to inject for app '%s'. Error: %s", diagAppID, err)
   monitoring.RecordFailedSidecarInjectionCount(diagAppID, "patch")
} else if len(patchOps) == 0 {
   // len(patchOps) == 0 表示什么都没改，返回  Allowed: true
   admissionResponse = &v1.AdmissionResponse{
      Allowed: true,
   }
} else {
   var patchBytes []byte
   // 将 patchOps 序列化为json
   patchBytes, err = json.Marshal(patchOps)
   if err != nil {
      admissionResponse = toAdmissionResponse(err)
   } else {
      // 返回AdmissionResponse
      admissionResponse = &v1.AdmissionResponse{
         Allowed: true,
         Patch:   patchBytes,
         PatchType: func() *v1.PatchType {
            pt := v1.PatchTypeJSONPatch
            return &pt
         }(),
      }
   }
}
```

组装 AdmissionReview:

```go
admissionReview := v1.AdmissionReview{}
if admissionResponse != nil {
   admissionReview.Response = admissionResponse
   if ar.Request != nil {
      admissionReview.Response.UID = ar.Request.UID
      admissionReview.SetGroupVersionKind(*gvk)
   }
}
```

将应答序列化并返回：

```go
log.Infof("ready to write response ...")
respBytes, err := json.Marshal(admissionReview)
if err != nil {
   http.Error(
      w,
      err.Error(),
      http.StatusInternalServerError,
   )

   log.Errorf("Sidecar injector failed to inject for app '%s'. Can't deserialize response: %s", diagAppID, err)
   monitoring.RecordFailedSidecarInjectionCount(diagAppID, "response")
}
w.Header().Set("Content-Type", "application/json")
if _, err := w.Write(respBytes); err != nil {
   log.Error(err)
} else {
   log.Infof("Sidecar injector succeeded injection for app '%s'", diagAppID)
   monitoring.RecordSuccessfulSidecarInjectionCount(diagAppID)
}
```

## 帮助类代码

### toAdmissionResponse方法

toAdmissionResponse 方法用于从一个 error 创建 k8s 的 AdmissionResponse ：

```go
// toAdmissionResponse is a helper function to create an AdmissionResponse
// with an embedded error
func toAdmissionResponse(err error) *v1.AdmissionResponse {
   return &v1.AdmissionResponse{
      Result: &metav1.Status{
         Message: err.Error(),
      },
   }
}
```

### 获取AppID

getAppIDFromRequest() 方法从 AdmissionRequest 中获取AppID：

```go
func getAppIDFromRequest(req *v1.AdmissionRequest) string {
   // default App ID
   appID := ""

   // if req is not given
   if req == nil {
      return appID
   }

   var pod corev1.Pod
   // 解析pod的raw数据为json
   if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
      log.Warnf("could not unmarshal raw object: %v", err)
   } else {
      // 然后从pod信息中获取appID
      appID = getAppID(pod)
   }

   return appID
}


```

getAppID()方法的实现如下，首先读取 "dapr.io/app-id" 的 Annotation，如果没有，则取 pod 的 name 作为默认AppID：

```go
const	appIDKey                          = "dapr.io/app-id"
func getAppID(pod corev1.Pod) string {
	return getStringAnnotationOrDefault(pod.Annotations, appIDKey, pod.GetName())
}
```

## 分支代码

### ServiceAccount 相关代码

AllowedControllersServiceAccountUID（）方法返回UID数组，这些是 webhook handler 上容许的 service account 列表：

```go
var allowedControllersServiceAccounts = []string{
	"replicaset-controller",
	"deployment-controller",
	"cronjob-controller",
	"job-controller",
	"statefulset-controller",
}

// AllowedControllersServiceAccountUID returns an array of UID, list of allowed service account on the webhook handler
func AllowedControllersServiceAccountUID(ctx context.Context, kubeClient *kubernetes.Clientset) ([]string, error) {
   allowedUids := []string{}
   for i, allowedControllersServiceAccount := range allowedControllersServiceAccounts {
      saUUID, err := getServiceAccount(ctx, kubeClient, allowedControllersServiceAccount)
      // i == 0 => "replicaset-controller" is the only one mandatory
      if err != nil && i == 0 {
         return nil, err
      } else if err != nil {
         log.Warnf("Unable to get SA %s UID (%s)", allowedControllersServiceAccount, err)
         continue
      }
      allowedUids = append(allowedUids, saUUID)
   }

   return allowedUids, nil
}

func getServiceAccount(ctx context.Context, kubeClient *kubernetes.Clientset, allowedControllersServiceAccount string) (string, error) {
	ctxWithTimeout, cancel := context.WithTimeout(ctx, getKubernetesServiceAccountTimeoutSeconds*time.Second)
	defer cancel()

	sa, err := kubeClient.CoreV1().ServiceAccounts(metav1.NamespaceSystem).Get(ctxWithTimeout, allowedControllersServiceAccount, metav1.GetOptions{})
	if err != nil {
		return "", err
	}

	return string(sa.ObjectMeta.UID), nil
}
```