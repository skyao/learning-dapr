---
type: docs
title: "Injector的代码实现"
linkTitle: "代码实现"
weight: 2102
date: 2021-01-31
description: >
  Dapr Injector的代码实现
---

## Inject的流程

以e2e中的 stateapp 为例。

### 应用的原始Deployment

`tests/apps/stateapp/service.yaml` 中是 stateapp 的 Service 定义和 Deployment定义。

 Service的定义没有什么特殊：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: stateapp
  labels:
    testapp: stateapp
spec:
  selector:
    testapp: stateapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
```

deployment的定义：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateapp
  labels:
    testapp: stateapp
spec:
  replicas: 1
  selector:
    matchLabels:
      testapp: stateapp
  template: # stateapp的pod定义
    metadata:
      labels:
        testapp: stateapp
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "stateapp"
        dapr.io/app-port: "3000"
    spec:   #stateapp的container定义，暂时pod中只定义了这个一个container
      containers:
      - name: stateapp
        image: docker.io/YOUR_DOCKER_ALIAS/e2e-stateapp:dev
        ports:
        - containerPort: 3000
        imagePullPolicy: Always

```

单独看 stateapp 的 pod 定义的 annotations ，

```yaml
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "stateapp"
        dapr.io/app-port: "3000"
```



## 源码

getPodPatchOperations：

```go
func (i *injector) getPodPatchOperations(ar *v1beta1.AdmissionReview,
	namespace, image string, kubeClient *kubernetes.Clientset, daprClient scheme.Interface) ([]PatchOperation, error) {
	req := ar.Request
	var pod corev1.Pod
	if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
		errors.Wrap(err, "could not unmarshal raw object")
		return nil, err
	}

	log.Infof(
		"AdmissionReview for Kind=%v, Namespace=%v Name=%v (%v) UID=%v "+
			"patchOperation=%v UserInfo=%v",
		req.Kind,
		req.Namespace,
		req.Name,
		pod.Name,
		req.UID,
		req.Operation,
		req.UserInfo,
	)

	if !isResourceDaprEnabled(pod.Annotations) || podContainsSidecarContainer(&pod) {
		return nil, nil
	}
  ...
```

这个info日志打印的例子如下：

```
{"instance":"dapr-sidecar-injector-5f6f4bb6df-n5dsk","level":"info","msg":"AdmissionReview for Kind=/v1, Kind=Pod, Namespace=dapr-tests Name= () UID=d0126a13-9efd-432e-894a-5ddbee55898c patchOperation=CREATE UserInfo={system:serviceaccount:kube-system:replicaset-controller 3e5de149-07a3-434e-a8de-209abee69760 [system:serviceaccounts system:serviceaccounts:kube-system system:authenticated] map[]}","scope":"dapr.injector","time":"2020-09-25T07:07:07.6482457Z","type":"log","ver":"edge"}
```

可以看到在 namespace dapr-tests 下 pod 有 CREATE operation时Injector有开始工作。

`isResourceDaprEnabled(pod.Annotations)` 检查是否是 dapr，判断的方式是看 pod 是否有名为`dapr.io/enabled` 的 annotation并且设置为true，缺省为false：

```go
const (
	daprEnabledKey                    = "dapr.io/enabled"
)
func isResourceDaprEnabled(annotations map[string]string) bool {
	return getBoolAnnotationOrDefault(annotations, daprEnabledKey, false)
}
```

podContainsSidecarContainer 检查 pod 是不是已经包含 dapr的sidecar，判断的方式是看 container 的名字是不是 `daprd`： 

```go
const (
	sidecarContainerName              = "daprd"
)
func podContainsSidecarContainer(pod *corev1.Pod) bool {
	for _, c := range pod.Spec.Containers {
		if c.Name == sidecarContainerName {
			return true
		}
	}
	return false
}
```

继续getPodPatchOperations()：

```go
	id := getAppID(pod)
	// Keep DNS resolution outside of getSidecarContainer for unit testing.
	placementAddress := fmt.Sprintf("%s:80", getKubernetesDNS(placementService, namespace))
	sentryAddress := fmt.Sprintf("%s:80", getKubernetesDNS(sentryService, namespace))
	apiSrvAddress := fmt.Sprintf("%s:80", getKubernetesDNS(apiAddress, namespace))
```

getAppID(pod) 通过读取 annotation 来获取应用id，注意 "dapr.io/id" 已经废弃，1.0 之后将被删除，替换为dapr.io/app-id"：

```go
const (
	appIDKey                          = "dapr.io/app-id"
  	// Deprecated, remove in v1.0
	idKey                 = "dapr.io/id"
)
func getAppID(pod corev1.Pod) string {
	id := getStringAnnotationOrDefault(pod.Annotations, appIDKey, "")
	if id != "" {
		return id
	}

	return getStringAnnotationOrDefault(pod.Annotations, idKey, pod.GetName())
}
```

#### mtlsEnabled的判断

```go
	var trustAnchors string
	var certChain string
	var certKey string
	var identity string

	mtlsEnabled := mTLSEnabled(daprClient)
	if mtlsEnabled {
		trustAnchors, certChain, certKey = getTrustAnchorsAndCertChain(kubeClient, namespace)
		identity = fmt.Sprintf("%s:%s", req.Namespace, pod.Spec.ServiceAccountName)
	}
```

mTLSEnabled判断的方式，居然是读取所有的namespace下的dapr configuration：

```go
const (
	// NamespaceAll is the default argument to specify on a context when you want to list or filter resources across all namespaces
	NamespaceAll string = ""
)
func mTLSEnabled(daprClient scheme.Interface) bool {
	resp, err := daprClient.ConfigurationV1alpha1().Configurations(meta_v1.NamespaceAll).List(meta_v1.ListOptions{})
	if err != nil {
		return defaultMtlsEnabled
	}

	for _, c := range resp.Items {
		if c.GetName() == defaultConfig {  // "daprsystem"
			return c.Spec.MTLSSpec.Enabled
		}
	}
	return defaultMtlsEnabled
}
```

通过读取k8s的资源来判断是否要开启 mtls，`tests/config/dapr_mtls_off_config.yaml` 有example内容：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprsystem # 名字一定要是 daprsystem
spec:
  mtls:
    enabled: "false"  # 在这里配置要不要开启 mtls
    workloadCertTTL: "1h"
    allowedClockSkew: "20m"
```



但这个坑货

```bash
E0925 09:37:53.480772       1 reflector.go:153] sigs.k8s.io/controller-runtime/pkg/cache/internal/informers_map.go:224: Failed to list *v1alpha1.Configuration: v1alpha1.ConfigurationList.Items: []v1alpha1.Configuration: v1alpha1.Configuration.Spec: v1alpha1.ConfigurationSpec.MTLSSpec: v1alpha1.MTLSSpec.Enabled: ReadBool: expect t or f, but found ", error found in #10 byte of ...|enabled":"false","wo|..., bigger context ...|pec":{"mtls":{"allowedClockSkew":"20m","enabled":"false","workloadCertTTL":"1h"}}},{"apiVersion":"da|...
```



### 生效的应用pod定义



```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    dapr.io/app-id: stateapp
    dapr.io/app-port: "3000"
    dapr.io/enabled: "true"
    dapr.io/sidecar-cpu-limit: "4.0"
    dapr.io/sidecar-cpu-request: "0.5"
    dapr.io/sidecar-memory-limit: 512Mi
    dapr.io/sidecar-memory-request: 250Mi
  creationTimestamp: "2020-09-25T07:07:07Z"
  generateName: stateapp-567b6b9c6f-
  labels:
    pod-template-hash: 567b6b9c6f
    testapp: stateapp
  name: stateapp-567b6b9c6f-84kzb
  namespace: dapr-tests
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: stateapp-567b6b9c6f
    uid: 25a34367-79ed-4e19-868a-5b063a45b1f4
  resourceVersion: "146616"
  selfLink: /api/v1/namespaces/dapr-tests/pods/stateapp-567b6b9c6f-84kzb
  uid: 0f4060df-0312-4d73-91c1-6f085462b33d
  spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
  containers:
  - env:
    - name: DAPR_HTTP_PORT
      value: "3500"
    - name: DAPR_GRPC_PORT
      value: "50001"
    image: docker.io/skyao/e2e-stateapp:dev-linux-amd64
    imagePullPolicy: Always
    name: stateapp
    ports:
    - containerPort: 3000
      name: http
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-qncjc
      readOnly: true
  - args:
    - --mode
    - kubernetes
    - --dapr-http-port
    - "3500"
    - --dapr-grpc-port
    - "50001"
    - --dapr-internal-grpc-port
    - "50002"
    - --app-port
    - "3000"
    - --app-id
    - stateapp
    - --control-plane-address
    - dapr-api.dapr-system.svc.cluster.local:80
    - --app-protocol
    - http
    - --placement-host-address
    - dapr-placement.dapr-system.svc.cluster.local:80
    - --config
    - ""
    - --log-level
    - info
    - --app-max-concurrency
    - "-1"
    - --sentry-address
    - dapr-sentry.dapr-system.svc.cluster.local:80
    - --metrics-port
    - "9090"
    - --enable-mtls
    command:
    - /daprd
    env:
    - name: DAPR_HOST_IP
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: status.podIP
    - name: NAMESPACE
      value: dapr-tests
    - name: DAPR_TRUST_ANCHORS
      value: |
        -----BEGIN CERTIFICATE-----
        MIIB3TCCAYKgAwIBAgIRAMra+wjgMY6ABDtu3/vJ0NcwCgYIKoZIzj0EAwIwMTEX
        MBUGA1UEChMOZGFwci5pby9zZW50cnkxFjAUBgNVBAMTDWNsdXN0ZXIubG9jYWww
        HhcNMjAwOTI1MDU1ODAzWhcNMjEwOTI1MDU1ODAzWjAxMRcwFQYDVQQKEw5kYXBy
        LmlvL3NlbnRyeTEWMBQGA1UEAxMNY2x1c3Rlci5sb2NhbDBZMBMGByqGSM49AgEG
        CCqGSM49AwEHA0IABE/w/8YBtRJPYNJkcDM05e9PhrbGjBU/RQd09J909OJebDe8
        rthysygWrcGYHYKziKK2Pyc1j4ua2xklLC5DFEWjezB5MA4GA1UdDwEB/wQEAwIC
        BDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwDwYDVR0TAQH/BAUwAwEB
        /zAdBgNVHQ4EFgQUQ2v6OiayM9V4DPAU6UZHGe/nc1swGAYDVR0RBBEwD4INY2x1
        c3Rlci5sb2NhbDAKBggqhkjOPQQDAgNJADBGAiEAtVBx9vDXiRE3fXJTU2yK11W5
        eo+Ce4+U6/vXDtzw4PUCIQDlLOB45ihOAhhLVLG9akhgwJOrgZLEW3FZjRabpSsb
        og==
        -----END CERTIFICATE-----
    - name: DAPR_CERT_CHAIN
      value: |
        -----BEGIN CERTIFICATE-----
        MIIBxDCCAWqgAwIBAgIQQ1sfEH4aYacFZwBau+aOozAKBggqhkjOPQQDAjAxMRcw
        FQYDVQQKEw5kYXByLmlvL3NlbnRyeTEWMBQGA1UEAxMNY2x1c3Rlci5sb2NhbDAe
        Fw0yMDA5MjUwNTU4MDNaFw0yMTA5MjUwNTU4MDNaMBgxFjAUBgNVBAMTDWNsdXN0
        ZXIubG9jYWwwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARhj7MQ1uiOkZvJ0AYV
        uiFca/Iu9D5O98E5JN1mjCohRawk+QT1PjW05YtmyVji4Tt6ckIMvOXwG3aoTsGO
        UbRio30wezAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4E
        FgQUTPUh0WWBB5baKs3aJjMzInVLX/EwHwYDVR0jBBgwFoAUQ2v6OiayM9V4DPAU
        6UZHGe/nc1swGAYDVR0RBBEwD4INY2x1c3Rlci5sb2NhbDAKBggqhkjOPQQDAgNI
        ADBFAiBO0oCadeYyLM+RkSAYPSTtjMyEZ0wv1/BsWuUMg+KZ6AIhALHnT0pxiqlj
        miYT4WZWvaBc17AbUh1efgV2DAaNKm54
        -----END CERTIFICATE-----
        
    - name: DAPR_CERT_KEY
      value: |
        -----BEGIN EC PRIVATE KEY-----
        MHcCAQEEIDj6niLJ5ep+fDdY71bKyWl9RZHrXyRjND6pWySL2Q4UoAoGCCqGSM49
        AwEHoUQDQgAEYY+zENbojpGbydAGFbohXGvyLvQ+TvfBOSTdZowqIUWsJPkE9T41
        tOWLZslY4uE7enJCDLzl8Bt2qE7BjlG0Yg==
        -----END EC PRIVATE KEY-----
    - name: SENTRY_LOCAL_IDENTITY
      value: default:dapr-tests
    image: docker.io/skyao/daprd:dev-linux-amd64
    imagePullPolicy: Always
    livenessProbe:
      failureThreshold: 3
      httpGet:
        path: /v1.0/healthz
        port: 3500
        scheme: HTTP
      initialDelaySeconds: 3
      periodSeconds: 6
      successThreshold: 1
      timeoutSeconds: 3
    name: daprd
    ports:
    - containerPort: 3500
      name: dapr-http
      protocol: TCP
    - containerPort: 50001
      name: dapr-grpc
      protocol: TCP
    - containerPort: 50002
      name: dapr-internal
      protocol: TCP
    - containerPort: 9090
      name: dapr-metrics
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /v1.0/healthz
        port: 3500
        scheme: HTTP
      initialDelaySeconds: 3
      periodSeconds: 6
      successThreshold: 1
      timeoutSeconds: 3
    resources:
      limits:
        cpu: "4"
        memory: 512Mi
      requests:
        cpu: 500m
        memory: 250Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-qncjc
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: docker-desktop
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-qncjc
    secret:
      defaultMode: 420
      secretName: default-token-qncjc
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T07:07:07Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T07:07:07Z"
    message: 'containers with unready status: [daprd]'
    reason: ContainersNotReady
    status: "False"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T07:07:07Z"
    message: 'containers with unready status: [daprd]'
    reason: ContainersNotReady
    status: "False"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T07:07:07Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://26a1d85ac6e2accd833832681b8dc2aa809e3c0fcfa293398bd5e7c2e8bf3e2b
    image: skyao/daprd:dev-linux-amd64
    imageID: docker-pullable://skyao/daprd@sha256:387f3bf4e7397c43dca9ac2d248a9ce790b1c1888aa0d6de3b07107ce124752f
    lastState:
      terminated:
        containerID: docker://26a1d85ac6e2accd833832681b8dc2aa809e3c0fcfa293398bd5e7c2e8bf3e2b
        exitCode: 1
        finishedAt: "2020-09-25T08:03:14Z"
        reason: Error
        startedAt: "2020-09-25T08:03:04Z"
    name: daprd
    ready: false
    restartCount: 21
    started: false
    state:
      waiting:
        message: back-off 5m0s restarting failed container=daprd pod=stateapp-567b6b9c6f-84kzb_dapr-tests(0f4060df-0312-4d73-91c1-6f085462b33d)
        reason: CrashLoopBackOff
  - containerID: docker://737745ace04213c9519ad1f91e248015c89a80e2b3d61081c3c530d1c89bdbae
    image: skyao/e2e-stateapp:dev-linux-amd64
    imageID: docker-pullable://skyao/e2e-stateapp@sha256:16351b331f1338a61348c9a87fce43728369f1bf18ee69d9d45fb13db0283644
    lastState: {}
    name: stateapp
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-09-25T07:07:24Z"
  hostIP: 192.168.65.3
  phase: Running
  podIP: 10.1.0.194
  podIPs:
  - ip: 10.1.0.194
  qosClass: Burstable
  startTime: "2020-09-25T07:07:07Z"
```





## 其他

### injector自身的pod定义

dapr-sidecar-injector

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/path: /
    prometheus.io/port: "9090"
    prometheus.io/scrape: "true"
  creationTimestamp: "2020-09-25T05:57:37Z"
  generateName: dapr-sidecar-injector-5f6f4bb6df-
  labels:
    app: dapr-sidecar-injector
    app.kubernetes.io/component: sidecar-injector
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/name: dapr
    app.kubernetes.io/part-of: dapr
    app.kubernetes.io/version: dev-linux-amd64
    pod-template-hash: 5f6f4bb6df
  name: dapr-sidecar-injector-5f6f4bb6df-n5dsk
  namespace: dapr-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: dapr-sidecar-injector-5f6f4bb6df
    uid: ff47b1df-6da7-4a19-b99d-15622ca3a485
  resourceVersion: "133143"
  selfLink: /api/v1/namespaces/dapr-system/pods/dapr-sidecar-injector-5f6f4bb6df-n5dsk
  uid: 40df3834-4df2-495a-aa26-5b2a22de7639
  spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - preference:
          matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
        weight: 1
  containers:
  - args:
    - --log-level
    - info
    - --log-as-json
    - --metrics-port
    - "9090"
    command:
    - /injector
    env:
    - name: TLS_CERT_FILE
      value: /dapr/cert/tls.crt
    - name: TLS_KEY_FILE
      value: /dapr/cert/tls.key
    - name: SIDECAR_IMAGE
      value: docker.io/skyao/daprd:dev-linux-amd64
    - name: NAMESPACE
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.namespace
    image: docker.io/skyao/dapr:dev-linux-amd64
    imagePullPolicy: Always
        livenessProbe:
      failureThreshold: 5
      httpGet:
        path: /healthz
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 3
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
    name: dapr-sidecar-injector
    ports:
    - containerPort: 4000
      name: https
      protocol: TCP
    - containerPort: 9090
      name: metrics
      protocol: TCP
    readinessProbe:
      failureThreshold: 5
      httpGet:
        path: /healthz
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 3
      periodSeconds: 3
      successThreshold: 1
      timeoutSeconds: 1
    resources: {}
        securityContext:
      runAsUser: 1000
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /dapr/cert
      name: cert
      readOnly: true
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: dapr-operator-token-lgpvc
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: docker-desktop
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: dapr-operator
  serviceAccountName: dapr-operator
  terminationGracePeriodSeconds: 30
  tolerations:
    - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: cert
    secret:
      defaultMode: 420
      secretName: dapr-sidecar-injector-cert
  - name: dapr-operator-token-lgpvc
    secret:
      defaultMode: 420
      secretName: dapr-operator-token-lgpvc
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T05:57:37Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T05:58:10Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T05:58:10Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-09-25T05:57:37Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://a820646b468a07eabdd89ca133f062a93e85256afc6c19c1bdf13b56980ec5e9
    image: skyao/dapr:dev-linux-amd64
    imageID: docker-pullable://skyao/dapr@sha256:77003eee9fd02d9fc24c2e9f385a6c86223bc35915cede98a8897c0dfc51ee61
    lastState: {}
    name: dapr-sidecar-injector
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-09-25T05:58:06Z"
  hostIP: 192.168.65.3
  phase: Running
  podIP: 10.1.0.188
  podIPs:
  - ip: 10.1.0.188
  qosClass: BestEffort
  startTime: "2020-09-25T05:57:37Z"
```

