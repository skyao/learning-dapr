---
title: "在容器中运行外部组件"
weight: 30
date: 2022-04-08
description: >
  在linux平台上通过容器运行daprd需要访问的外部组件
---

本地开发时，各个组件只需要使用最简单的部署方案即可。

## redis

执行下面命令，指定端口映射和访问密码：

```bash
docker run -d -p 6379:6379 redis --requirepass "abc123"
```

可以看到后台启动的 redis 进行：

```bash
ps -ef | grep redis
999         3348    3327  0 14:18 ?        00:00:12 redis-server *:6379
```

然后就可以配置基于 redis 的 component，如 state：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  initTimeout: 1m
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: abc123
  - name: actorStateStore
    value: true
```



## kafka

kafka 相对 redis 要麻烦一些，因为 kafka 需要 zookeeper。因此需要使用 docker-compose 工具来启动多个 container。

https://github.com/conduktor/kafka-stack-docker-compose

这里有配置好的多种 kafka 方案，我们可以选择最简单的一种，`zk-single-kafka-single`

```bash
$ wget https://github.com/conduktor/kafka-stack-docker-compose/blob/master/zk-single-kafka-single.yml
$ docker-compose -f zk-single-kafka-single.yml up -d

# 可以看到 kafka 和 zeekeeper 在运行
$ ps -ef | grep kafka 
sky         3172    2436  0 18:05 pts/0    00:00:00 /usr/bin/python3 /usr/bin/docker-compose -f zk-single-kafka-single.yml up

sky         3930    3908  1 18:06 ?        00:00:04 java -Xmx512M -Xms512M -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true -Xlog:gc*:file=/var/log/kafka/zookeeper-gc.log:time,tags:filecount=10,filesize=100M -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dkafka.logs.dir=/var/log/kafka -Dlog4j.configuration=file:/etc/kafka/log4j.properties -cp /usr/bin/../share/java/kafka/*:/usr/bin/../share/java/confluent-telemetry/* org.apache.zookeeper.server.quorum.QuorumPeerMain /etc/kafka/zookeeper.properties

sky         4114    4090  3 18:06 ?        00:00:10 java -Xmx1G -Xms1G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true -Xlog:gc*:file=/var/log/kafka/kafkaServer-gc.log:time,tags:filecount=10,filesize=100M -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.rmi.port=9999 -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.port=9999 -Dkafka.logs.dir=/var/log/kafka -Dlog4j.configuration=file:/etc/kafka/log4j.properties -cp /usr/bin/../share/java/kafka/*:/usr/bin/../share/java/confluent-telemetry/* kafka.Kafka /etc/kafka/kafka.properties
```

kafka 和 zookeeper 运行的端口信息可以通过下面的命令看到：

```bash
$ docker-compose -f zk-single-kafka-single.yml ps
 Name             Command            State                                         Ports                                       
-------------------------------------------------------------------------------------------------------------------------------
kafka1   /etc/confluent/docker/run   Up      0.0.0.0:9092->9092/tcp,:::9092->9092/tcp, 0.0.0.0:9999->9999/tcp,:::9999->9999/tcp
zoo1     /etc/confluent/docker/run   Up      0.0.0.0:2181->2181/tcp,:::2181->2181/tcp, 2888/tcp, 3888/tcp 
```

kafka 运行在 `localhost:9092`。

然后就可以配置基于 kafka 的 component，如 pubsub：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: localhost:9092
  - name: consumerGroup
    value: pubsubgroup1
  - name: authRequired
    value: "false"
  - name: initialOffset
    value: oldest
  - name: disableTls
    value: true
```





参考： https://www.conduktor.io/kafka/how-to-start-kafka-using-docker
