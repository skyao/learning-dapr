---
title: "运行性能测试"
linkTitle: "运行性能测试"
weight: 10
date: 2021-05-09
description: >
  运行Dapr的性能测试
---



## 准备工作

基本类似 e2e 测试。

### 设置环境变量

首先要设置相关的环境变量。

#### amd64

在 amd64 机器上：

```bash
export DAPR_REGISTRY=docker.io/skyao
export DAPR_TAG=dev
export DAPR_NAMESPACE=dapr-tests
export TARGET_OS=linux
export TARGET_ARCH=amd64
export GOOS=linux
export GOARCH=amd64
export DAPR_TEST_NAMESPACE=dapr-tests
export DAPR_TEST_REGISTRY=docker.io/skyao
export DAPR_TEST_TAG=dev-linux-amd64
export DAPR_TEST_MINIKUBE_IP=192.168.100.40		# use this in IDE
export MINIKUBE_NODE_IP=192.168.100.40 			# use this in make command
```

### 构建并部署dapr到k8s中

```bash
$ make create-test-namespace

$ make build-linux
$ make docker-build
$ make docker-push

$ make docker-deploy-k8s
```

如果之前有过部署，则需要在重新部署前删除之前的部署，有两种情况：

1. 只清除 dapr 的控制平面

   ```bash
   $ helm uninstall dapr  -n dapr-tests
   $ make docker-deploy-k8s
   ```

2. 清除所有 dapr 内容

   ```bash
   $ helm uninstall dapr  -n dapr-tests
   $ make delete-test-namespace 
   # 再重复上面的构建和部署过程
   ```

类似 e2e 测试，性能测试中本有一个额外的操作，实际是安装 redis / kafka / mongodb ：

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

对于性能测试，没有必要，可以视情况（是否要测试 actor 相关的功能）看是否要安装 redis：

```bash
make setup-test-env-redis 
```

### dapr相关的设置

关闭遥测：

```bash
$ make setup-app-configurations

kubectl apply -f ./tests/config/dapr_observability_test_config.yaml --namespace dapr-tests
configuration.dapr.io/disable-telemetry created
configuration.dapr.io/obs-defaultmetric created
```

关闭mtls：

```bash
$ make setup-disable-mtls
kubectl apply -f ./tests/config/dapr_mtls_off_config.yaml --namespace dapr-tests
configuration.dapr.io/daprsystem created
```

准备测试用的components：

```bash
# 切记不要用这个命令
$ make setup-test-components
```

这个命令会将 `tests/config` 下的yaml文件都安装到k8s下，有些多，而且部分component文件在没有配置好外部组件时会导致 daprd 启动失败。安全起见，如果只是跑个别性能测试的test case，手工安装需要的就好了

```bash
# 对于 actor 相关的性能测试案例，需要安装 redis 并开启 statestore-actos
# 由于redis 配置中使用到了 secret，因此需要安装 kubernetes_redis_secret
# make setup-test-env-redis 
$ k apply -f ./tests/config/dapr_redis_state_actorstore.yaml -n dapr-tests
$ k apply -f ./tests/config/kubernetes_redis_secret.yaml -n dapr-tests

# 对于 pubsub 的测试，只需要开启 in-memory pubsub
$ k apply -f ./tests/config/dapr_in_memory_pubsub.yaml -n dapr-tests

# 对于 state 的测试，只需要开启 in-memory state
$ k apply -f ./tests/config/dapr_in_memory_state.yaml -n dapr-tests
```

## 准备测试用的应用

```bash
$ make build-perf-app-all
$ make push-perf-app-all
```

如果是在开发或者修改某一个测试案例，要节约时间，不需要构建和发布所有的测试案例。只单独构建和推送某一个性能测试的应用，可以直接调用 make target，如针对 service_invocation_http 这个 test case :

```bash
$ make build-perf-app-service_invocation_http
$ make push-perf-app-service_invocation_http
```

## 执行性能测试

测试前先安装一下 jq，后面会用到：

```bash
brew install jq
=======


## 运行性能测试
# 切记不要跑 test-perf-all，由于环境变量在各个perf test case中的设置要求不同，全部一起跑会有问题。
# 在本地（无论是IDE还是终端）不要跑 test-perf-all，只能单独跑某一个 perf test case
# make test-perf-all
make test-perf-xxxxx
# 特别注意，如果没有设置性能测试输入条件相关的环境变量，直接默认跑，有一些 perf test case 是可以跑起来的
make test-perf-state_get_http
make test-perf-state_get_grpc
make test-perf-service_invoke_http
make test-perf-service_invoke_grpc
make test-perf-pubsub_publish_grpc

# 有部分 perf test 是跑不起来的，会报错
make test-perf-actor_timer. # fortio报错，HTTP响应为 405 Method Not Allow，必须用 HTTP POST

```


跑的时候特别注意日志文件中的这些内容：

```bash
2022/03/25 18:02:05 Installing test apps...
2022/03/25 18:02:09 Adding app {testapp 0  map[] true perf-service_invocation_http:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
2022/03/25 18:02:09 Adding app {tester 3001  map[] true perf-tester:dev-linux-amd64  docker.io/skyao 1 true true   4.0 0.1 800Mi 2500Mi 4.0 0.1 512Mi 250Mi <nil> false}
```

仔细检查 app 的镜像信息，包括 tag 要求是 "dev-linux-amd64", image registry 要求是自己设定的类似 "docker.io/skyao"，否则 pod 会无法启动。

最好先 `env | grep DAPR` 看一下设置的环境变量是否有生效。



## 本地debug

perf test 的测试案例都是用 go test 编写，原则上只要前面步骤准备好，是可以在本地 IDE 中以 debug 方式启动 perf test 的测试案例，然后进行 debug 的。

> 特别注意：actor 相关的 test case 要设置好性能测试输入条件的环境变量

## 特殊情况

### 彻底清理namespace

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

否则，重新安装 dapr 控制面时，如果 dapr 控制面的 namespace 发生变化，就会出报错：

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

