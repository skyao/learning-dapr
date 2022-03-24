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

在 m1 macbook 上构建，运行时报错：

```bash
$ k get pods -A
NAMESPACE     NAME                                     READY   STATUS             RESTARTS   AGE
dapr-tests    testapp-85d8d9db89-p6mms                 0/2     CrashLoopBackOff   13         11m

$ k logs testapp-85d8d9db89-p6mms -n dapr-tests testapp
standard_init_linux.go:228: exec user process caused: exec format error
```

没办法，换普通pc机器再试。dapr 对 m1 的支持是真的不好。

