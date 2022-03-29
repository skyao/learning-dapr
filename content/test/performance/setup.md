---
title: "运行性能测试"
linkTitle: "运行性能测试"
weight: 40
date: 2021-05-09
description: >
  运行Dapr的性能测试
---



## 准备工作

基本类似 e2e 测试，

```bash
$ make create-test-namespace

$ make build-linux
$ make docker-build
$ make docker-push

$ make docker-deploy-k8s

```

性能测试有一个额外的操作，实际是安装 redis / kafka / mongodb ：

```bash
$ make setup-3rd-party

helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" already exists with the same configuration, skipping
helm repo add stable https://charts.helm.sh/stable
"stable" already exists with the same configuration, skipping
helm repo add incubator https://charts.helm.sh/incubator
"incubator" already exists with the same configuration, skipping
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "incubator" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
helm install dapr-redis bitnami/redis --wait --timeout 5m0s --namespace dapr-tests -f ./tests/config/redis_override.yaml
......
helm install dapr-kafka bitnami/kafka -f ./tests/config/kafka_override.yaml --namespace dapr-tests --timeout 10m0s
......
helm install dapr-mongodb bitnami/mongodb -f ./tests/config/mongodb_override.yaml --namespace dapr-tests --wait --timeout 5m0s

```



```bash
$ make setup-app-configurations

kubectl apply -f ./tests/config/dapr_observability_test_config.yaml --namespace dapr-tests
configuration.dapr.io/disable-telemetry created
configuration.dapr.io/obs-defaultmetric created
```



```bash
$ make setup-disable-mtls
kubectl apply -f ./tests/config/dapr_mtls_off_config.yaml --namespace dapr-tests
configuration.dapr.io/daprsystem created
```



```bash
$ make setup-test-components
```

## 准备测试用的应用

```bash
$ make build-perf-app-all

$ make push-perf-app-all
```


备注：发现 `make build-perf-app-all` 时在m1 macbook上构建出来的测试应用的二进制文件有问题，在k8s上会出错。

## 执行性能测试

测试前先安装一下 jq，后面会用到：

```bash
brew install jq
=======


## 运行性能测试

```bash
make test-perf-all
```


特别注意日志文件中的这些内容：

```bash
2022/03/25 18:02:05 Installing test apps...
2022/03/25 18:02:09 Adding app {testapp 0  map[] true perf-service_invocation_http:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/25 18:02:09 Adding app {tester 3001  map[] true perf-tester:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
```

仔细检查 app 的镜像信息，包括 tag 要求是 "dev-linux-amd64", image registry 要求是自己设定的类似 "docker.io/skyao"，否则 pod 会无法启动。



## 本地debug性能测试



```bash
export DAPR_TEST_REGISTRY=docker.io/skyao
export DAPR_TEST_TAG=dev-linux-amd64
```

在文件 `tests/runner/kube_testplatform.go` 中获取 registry 的代码实现如下，取的是 DAPR_TEST_REGISTRY， 而不是 DAPR_REGISTRY：

```go
func (c *KubeTestPlatform) imageRegistry() string {
	reg := os.Getenv("DAPR_TEST_REGISTRY")
	if reg == "" {
		return defaultImageRegistry
	}
	return reg
}
```





## 特殊情况



在 m1 macbook 上构建，运行时报错：
=======
## 特殊情况

在 m1 macbook 上构建，运行 `make test-perf-all` 时报错：
```bash
$ k get pods -A
NAMESPACE     NAME                                     READY   STATUS             RESTARTS   AGE
dapr-tests    testapp-85d8d9db89-p6mms                 0/2     CrashLoopBackOff   13         11m

$ k logs testapp-85d8d9db89-p6mms -n dapr-tests testapp
standard_init_linux.go:228: exec user process caused: exec format error
```

没办法，换普通pc机器再试。dapr 对 m1 的支持是真的不好。

```bash
$ make build-perf-app-service_invocation_http
docker build -f ./tests/apps/perf/service_invocation_http/Dockerfile ./tests/apps/perf/service_invocation_http/. -t docker.io/skyao/perf-service_invocation_http:dev-linux-amd64

 => [stage-1 1/3] FROM docker.io/library/debian:buster-slim@sha256:bbf8ca5a94fe10b78b681d0f4efe8dbc23839d26e811ab6a1f252c7663c7e244 
 => CACHED [build_env 2/4] WORKDIR /app                                                                                                                                                                                            
 => CACHED [build_env 3/4] COPY app.go go.mod ./  
 => CACHED [build_env 4/4] RUN go get -d -v && go build -o app .   
 => CACHED [stage-1 2/3] COPY --from=build_env /app/app /                                                                                                                                                                          
 => exporting to image   
 => => exporting layers    
 => => writing image sha256:e2a015c035b224c846ec29773d5da68c765df0cf4747d78e648d52c1ed61970e    
 => => naming to docker.io/skyao/perf-service_invocation_http:dev-linux-amd64 
```

应该是 `go build -o app .` 这里的问题，没有加上 `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 `

修改代码，增加GOOS=linux GOARCH=amd64， 验证通过。提交的PR为：

[[m1 support\]Improve go build in dockerfile to support build docker images on m1 macbook · Issue #4426 · dapr/dapr (github.com)](https://github.com/dapr/dapr/issues/4426)

### 彻底清理

还要清理以下在 default namespace 中保存的内容：

```bash
k delete clusterrole dapr-operator-admin -n default
k delete clusterrole dashboard-reader -n default
k delete clusterrolebindings.rbac.authorization.k8s.io dapr-operator -n default
k delete clusterrolebindings.rbac.authorization.k8s.io dapr-role-tokenreview-binding -n default
k delete clusterrolebindings.rbac.authorization.k8s.io dashboard-reader-global -n default
k delete role secret-reader -n default
k delete rolebinding dapr-secret-reader -n default
k delete mutatingwebhookconfiguration dapr-sidecar-injector -n default
```

否则，重新安装dapr 控制面时，如果 dapr 控制面的 namespace 发生变化，就会出报错：

```bash
make docker-deploy-k8s

Deploying docker.io/skyao/dapr:dev to the current K8S context...
helm install \
                dapr --namespace=dapr-tests --wait --timeout 5m0s \
                --set global.ha.enabled=false --set-string global.tag=dev-linux-amd64 \
                --set-string global.registry=docker.io/skyao --set global.logAsJson=true \
                --set global.daprControlPlaneOs=linux --set global.daprControlPlaneArch=amd64 \
                --set dapr_placement.logLevel=debug --set dapr_sidecar_injector.sidecarImagePullPolicy=Always \
                --set global.imagePullPolicy=Always --set global.imagePullSecrets= \
                --set global.mtls.enabled=true \
                --set dapr_placement.cluster.forceInMemoryLog=true ./charts/dapr
Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: RoleBinding "dapr-secret-reader" in namespace "default" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "dapr-tests": current value is "dapr-system"
make: *** [docker-deploy-k8s] Error 1
```

