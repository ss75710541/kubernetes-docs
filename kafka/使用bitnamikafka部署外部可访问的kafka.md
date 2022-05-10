# 使用bitnami/kafka部署外部可访问的 kafka

## 安装 zookeeper

参考：[helm线下部署zookeeper.md](https://github.com/paradeum-team/operator-env/blob/main/zookeeper-operator/helm%E7%BA%BF%E4%B8%8B%E9%83%A8%E7%BD%B2zookeeper.md)

## 下载kafka chart

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm update
mkdir ~/bitnami-kafka
cd bitnami-kafka
helm pull bitnami/kafka
```

## 创建 values.yaml

注意修改 values 中 内网 IP、外网 IP、zookeeper 连接地址

```
global:
  storageClass: local-path
nodeSelector:
  kafka: "true"

zookeeper:
  enabled: false

hostNetwork: true

config: |-
  broker.id=-1
  inter.broker.listener.name=INTERNAL
  listener.security.protocol.map=EXTERNAL:PLAINTEXT,INTERNAL:PLAINTEXT
  listeners=INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:9093
  advertised.listeners=INTERNAL://内网IP:9092,EXTERNAL://外网IP:9093
  num.network.threads=3
  num.io.threads=8
  socket.send.buffer.bytes=102400
  socket.receive.buffer.bytes=102400
  socket.request.max.bytes=104857600
  log.dirs=/bitnami/kafka/data
  num.partitions=1
  num.recovery.threads.per.data.dir=1
  offsets.topic.replication.factor=1
  transaction.state.log.replication.factor=1
  transaction.state.log.min.isr=1
  log.flush.interval.messages=10000
  log.flush.interval.ms=1000
  log.retention.hours=168
  log.retention.bytes=1073741824
  log.segment.bytes=1073741824
  log.retention.check.interval.ms=300000
  zookeeper.connect=kafka-zk-zookeeper-headless.zookeeper.svc:2181
  zookeeper.connection.timeout.ms=6000
  group.initial.rebalance.delay.ms=0
```

## 执行安装

```
helm upgrade --install kafka kafka-16.2.10.tgz  -n kafka -f values.yaml
```

## 参考

https://artifacthub.io/packages/helm/bitnami/kafka

https://blog.csdn.net/lidelin10/article/details/105316252