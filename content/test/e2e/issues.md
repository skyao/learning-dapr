---
title: "搭建e2e测试环境时的常见问题"
linkTitle: "常见问题"
weight: 200
date: 2022-04-07
description: >
  搭建Dapr e2e测试环境时的常见问题
---



## mongodb 部署失败

部署测试用的 mongodb 失败，长时间挂住，然后超时报错：

```bash
$ make setup-test-env-mongodb 
helm install dapr-mongodb bitnami/mongodb -f ./tests/config/mongodb_override.yaml --namespace dapr-tests --wait --timeout 5m0s

Error: INSTALLATION FAILED: timed out waiting for the condition
make: *** [tests/dapr_tests.mk:269: setup-test-env-mongodb] Error 1
```

pod 状态一直是 Pending：

```bash
$ k get pods -A
NAMESPACE     NAME                                     READY   STATUS    RESTARTS       AGE
dapr-tests    dapr-mongodb-77fdf49576-d9lx4            0/1     Pending   0              8m37s
```

pod的描述提示是 PersistentVolumeClaims 的问题：

```bash
$ k describe pods dapr-mongodb-77fdf49576-d9lx4 -n dapr-tests
Name:           dapr-mongodb-77fdf49576-d9lx4

Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  8m55s                  default-scheduler  0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims.
  Warning  FailedScheduling  6m49s (x1 over 7m49s)  default-scheduler  0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims.
```

pvc 也是 pending 状态：

```bashk get pvc -n dapr-tests  
k get pvc -n dapr-tests  
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
dapr-mongodb   Pending                                                     22m
```

详细状态：

```bash
$ k describe pvc dapr-mongodb -n dapr-tests                  
Name:          dapr-mongodb
Namespace:     dapr-tests
StorageClass:  
Status:        Pending
Volume:        
Labels:        app.kubernetes.io/component=mongodb
               app.kubernetes.io/instance=dapr-mongodb
               app.kubernetes.io/managed-by=Helm
               app.kubernetes.io/name=mongodb
               helm.sh/chart=mongodb-11.1.5
Annotations:   meta.helm.sh/release-name: dapr-mongodb
               meta.helm.sh/release-namespace: dapr-tests
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       dapr-mongodb-77fdf49576-d9lx4
Events:
  Type    Reason         Age                   From                         Message
  ----    ------         ----                  ----                         -------
  Normal  FailedBinding  3m34s (x82 over 23m)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
```

"StorageClass" 为空，所以失败。 



参考资料：

- https://netapp-trident.readthedocs.io/en/stable-v18.07/kubernetes/operations/tasks/storage-classes.html
- https://blog.zuolinux.com/2020/06/10/nfs-client-provisioner.html

### sidecar 无法注入

一般是发生在重新安装 dapr 之后，主要原因是直接执行了 `kubectl delete namespace dapr-tests` ，而没有先执行 `helm uninstall dapr -n dapr-tests`

详细说明见：

https://github.com/dapr/dapr/issues/4612

### daprd启动时 sentry 证书报错

daprd启动时报错退出，日志如下：

```bash
time="2022-12-04T01:35:44.198741493Z" level=info msg="sending workload csr request to sentry" app_id=service-a instance=service-a-8bbd5bf88-fxc6k scope=dapr.runtime.grpc.internal type=log ver=edge
time="2022-12-04T01:35:44.207675954Z" level=fatal msg="failed to start internal gRPC server: error from authenticator CreateSignedWorkloadCert: error from sentry SignCertificate: rpc error: code = Unknown desc = error validating requester identity: csr validation failed: token/id mismatch. received id: dapr-tests:default" app_id=service-a instance=service-a-8bbd5bf88-fxc6k scope=dapr.runtime type=log ver=edge
```

这是没有执行 `make setup-disable-mtls` 的原因。

