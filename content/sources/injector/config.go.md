---
type: docs
title: "config.go的源码学习"
linkTitle: "config.go"
weight: 20020
date: 2021-04-19
description: >
  Dapr Injector 的 config 代码
---

Dapr injector package中的 config.go 文件的源码分析。

## 代码实现

### Config 结构体定义

Injector 相关的配置项

```go
// Config represents configuration options for the Dapr Sidecar Injector webhook server
type Config struct {
	TLSCertFile            string `envconfig:"TLS_CERT_FILE" required:"true"`
	TLSKeyFile             string `envconfig:"TLS_KEY_FILE" required:"true"`
	SidecarImage           string `envconfig:"SIDECAR_IMAGE" required:"true"`
	SidecarImagePullPolicy string `envconfig:"SIDECAR_IMAGE_PULL_POLICY"`
	Namespace              string `envconfig:"NAMESPACE" required:"true"`
}
```

### NewConfigWithDefaults() 方法

只设置了一个 SidecarImagePullPolicy 的默认值：

```go
func NewConfigWithDefaults() Config {
	return Config{
		SidecarImagePullPolicy: "Always",
	}
}
```

这个方法只被下面的 GetConfigFromEnvironment() 方法调用。

### GetConfigFromEnvironment() 方法

从环境中获取配置

```go
func GetConfigFromEnvironment() (Config, error) {
	c := NewConfigWithDefaults()
	err := envconfig.Process("", &c)
	return c, err
}
```

envconfig.Process() 的代码实现会通过反射读取到 Config 结构体的信息，然后根据设定的环境变量名来读取。

这个方法的调用只有一个地方，在injector main 函数的开始位置：

```go
func main() {
   log.Infof("starting Dapr Sidecar Injector -- version %s -- commit %s", version.Version(), version.Commit())

   ctx := signals.Context()
   cfg, err := injector.GetConfigFromEnvironment()
   if err != nil {
      log.Fatalf("error getting config: %s", err)
   }
   ......  
}
```

通过命令如 `k describe pod dapr-sidecar-injector-6f656b7dd-sg87p -n dapr-system` 拿到 injector pod 的yaml 文件，可以看到 Environment 的这一段：

```yaml
    Environment:
      TLS_CERT_FILE:              /dapr/cert/tls.crt
      TLS_KEY_FILE:               /dapr/cert/tls.key
      SIDECAR_IMAGE:              docker.io/skyao/daprd:dev-linux-amd64
      SIDECAR_IMAGE_PULL_POLICY:  IfNotPresent
      NAMESPACE:                  dapr-system (v1:metadata.namespace)
```





### injector yaml 备用

以下是完整的 injector yaml，留着备用：

```yaml
Name:         dapr-sidecar-injector-6f656b7dd-sg87p
Namespace:    dapr-system
Priority:     0
Node:         docker-desktop/192.168.65.3
Start Time:   Mon, 19 Apr 2021 15:04:07 +0800
Labels:       app=dapr-sidecar-injector
              app.kubernetes.io/component=sidecar-injector
              app.kubernetes.io/managed-by=helm
              app.kubernetes.io/name=dapr
              app.kubernetes.io/part-of=dapr
              app.kubernetes.io/version=dev-linux-amd64
              pod-template-hash=6f656b7dd
Annotations:  prometheus.io/path: /
              prometheus.io/port: 9090
              prometheus.io/scrape: true
Status:       Running
IP:           10.1.2.162
IPs:
  IP:           10.1.2.162
Controlled By:  ReplicaSet/dapr-sidecar-injector-6f656b7dd
Containers:
  dapr-sidecar-injector:
    Container ID:  docker://544dabf00bdaba9cf8f320218dd0b7e6d2ebce7fbf5184ce162d58bc693162d9
    Image:         docker.io/skyao/dapr:dev-linux-amd64
    Image ID:      docker-pullable://skyao/dapr@sha256:b4843ee78eabf014e15749bc4daa5c249ce3d33f796a89aaba9d117dd3dc76c9
    Ports:         4000/TCP, 9090/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /injector
    Args:
      --log-level
      info
      --log-as-json
      --enable-metrics
      --metrics-port
      9090
    State:          Running
      Started:      Mon, 19 Apr 2021 15:04:08 +0800
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:8080/healthz delay=3s timeout=1s period=3s #success=1 #failure=5
    Readiness:      http-get http://:8080/healthz delay=3s timeout=1s period=3s #success=1 #failure=5
    Environment:
      TLS_CERT_FILE:              /dapr/cert/tls.crt
      TLS_KEY_FILE:               /dapr/cert/tls.key
      SIDECAR_IMAGE:              docker.io/skyao/daprd:dev-linux-amd64
      SIDECAR_IMAGE_PULL_POLICY:  IfNotPresent
      NAMESPACE:                  dapr-system (v1:metadata.namespace)
    Mounts:
      /dapr/cert from cert (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from dapr-operator-token-cjpnd (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  cert:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  dapr-sidecar-injector-cert
    Optional:    false
  dapr-operator-token-cjpnd:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  dapr-operator-token-cjpnd
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  17m   default-scheduler  Successfully assigned dapr-system/dapr-sidecar-injector-6f656b7dd-sg87p to docker-desktop
  Normal  Pulled     17m   kubelet            Container image "docker.io/skyao/dapr:dev-linux-amd64" already present on machine
  Normal  Created    17m   kubelet            Created container dapr-sidecar-injector
  Normal  Started    17m   kubelet            Started container dapr-sidecar-injector
```

