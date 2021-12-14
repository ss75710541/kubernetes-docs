 # banzaicloud-stable/kafka-operator+local-path迁移主机

## 主机规划

| 主机名   | ip             | node label   |
| -------- | -------------- | ------------ |
| Logging1 | 172.16.13.77   | logging=true |
| Logging2 | 172.16.36.25   | logging=true |
| Logging3 | 172.16.115.194 | logging=true |
| kafka1   | 172.16.230.153 | kafka=true   |
| Kafka2   | 172.16.53.28   | kafka=true   |
| Kafka3   | 172.16.32.59   | kafka=true   |

## 迁移kafka 数据

### 备份pvc pv yaml

#### 查看pvc

```shell
kubectl get pvc

# 显示如下
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
kafka-0-storage-0-4ngpp   Bound    pvc-38dc6d2b-b612-445a-9a53-77903380a91f   10Gi       RWO            local-path     53d
kafka-1-storage-0-5hqjr   Bound    pvc-083d1a6c-dc2c-4ec1-bb18-d4008ee6f0b8   10Gi       RWO            local-path     53d
kafka-2-storage-0-9t4zd   Bound    pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f   10Gi       RWO            local-path     53d
```

#### 备份pvc pv yaml

```shell
kubectl get pvc kafka-0-storage-0-4ngpp -o yaml > kafka0.pvc.yaml
kubectl get pvc kafka-1-storage-0-5hqjr -o yaml > kafka1.pvc.yaml
kubectl get pvc kafka-2-storage-0-9t4zd -o yaml > kafka2.pvc.yaml
kubectl get pv pvc-38dc6d2b-b612-445a-9a53-77903380a91f -o yaml > kafka0.pv.yaml
kubectl get pv pvc-083d1a6c-dc2c-4ec1-bb18-d4008ee6f0b8 -o yaml > kafka1.pv.yaml
kubectl get pv pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f -o yaml > kafka2.pv.yaml
```

查看pv yaml  得知pv 所在主机及目录分别为：

| 主机     | kafka pod 编号 | 目录                                                         |
| :------- | -------------- | ------------------------------------------------------------ |
| Logging1 | kafka-2        | /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd |
| Logging2 | kafka-1        | /data/local-path-provisioner/pvc-083d1a6c-dc2c-4ec1-bb18-d4008ee6f0b8_kafka_kafka-1-storage-0-5hqjr |
| Logging3 | kafka-0        | /data/local-path-provisioner/pvc-38dc6d2b-b612-445a-9a53-77903380a91f_kafka_kafka-0-storage-0-4ngpp |

新规划的 kafka pv 主机及目录

| 主机   | kafka pod 编号 | 目录                                                         |
| :----- | -------------- | ------------------------------------------------------------ |
| Kafka1 | kafka-0        | /data/local-path-provisioner/pvc-38dc6d2b-b612-445a-9a53-77903380a91f_kafka_kafka-0-storage-0-4ngpp |
| Kafka2 | kafka-1        | /data/local-path-provisioner/pvc-083d1a6c-dc2c-4ec1-bb18-d4008ee6f0b8_kafka_kafka-1-storage-0-5hqjr |
| Kafka3 | kafka-2        | /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd |

### 迁移pod kafka-2 pv 数据

#### 修改helm 安装values.yaml,注释broker id 2 的内容

```yaml
  brokers:
    - id: 0
      brokerConfigGroup: "default"
      brokerConfig:
        nodePortExternalIP:
          external: "172.16.152.20" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
    - id: 1
      brokerConfigGroup: "default"
      brokerConfig:
        nodePortExternalIP:
          external: "172.16.142.252" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
    #- id: 2
    #  brokerConfigGroup: "default"
    #  brokerConfig:
    #    nodePortExternalIP:
    #     external: "172.16.233.132" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
```

#### 更新kafka cluster 

```shell
kubectl apply -f kafkacluster-with-nodeport-external.yaml
```

#### 删除对应broker pod

因为删除了broker 2 的配置, 所以删除kafka-2 pod 不会触发operator 新建pod

```shell
kubectl delete pod kafka-2-g6wjd
```

#### 在kafka3 主机复制pod kafka-2 pv 数据到新kafka3 主机 

```shell
mkdir -p /data/local-path-provisioner
scp -r 172.16.13.77:/data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd.bak
```

#### 删除kafka2 pvc 

因为是local-path storageclass 创建pvc , pv 自动删除

```shell
kubectl delete pvc kafka-2-storage-0-9t4zd
```

#### 重建 kafka2  pv

```shell
cp kafka2.pv.yaml kafka2.new.pv.yaml
```

编辑 kafka2.new.pv.yaml， 删除uid / resourceVersion/ creationTimestamp/claimRef/status信息, 主机名信息 logging1 替换为 kafka3

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kafka3.solarfs.k8s
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  volumeMode: Filesystem
```

```shell
kubectl apply -f kafka2.new.pv.yaml
```

#### 重建 kafka2 pvc

```shell
cp kafka2.pvc.yaml kafka2.new.pvc.yaml
```

编辑 kafka2.new.pvc.yaml， 删除uid (只保留ownerReferences中uid) / resourceVersion/ creationTimestamp/status信息, 主机名信息 logging1 替换为 kafka3

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    banzaicloud.com/last-applied: UEsDBBQACAAIAAAAAAAAAAAAAAAAAAAAAAAIAAAAb3JpZ2luYWxMkU+P1DAMxb+Lz8luu3/K0usiIYRg0R7ggEbITdwhahoXxwGJUb87SjuD5pY47/3ybJ9gJkWPitCfAFNiRQ2ccr3OXJJ+Qf0JPdxOOE5oIx8zrAaOlEhQ6TPOBD3sj3c2KwseyTYWDEQcKG4gXJaLCAwMwhPJBw893IHZyz+c/FesBhLOlBd0dGXjP4nklUYSSo4y9N8rOHwlyYHTRXgzYPqLwUUu/ibw7e92IMW2fhvZTS8V8o4i6eZRKWTAcVLhGEkulSmkGu9jJT7HkpUE9lRXgUqomrHz/ukN3tuH5nG0D+29s/jk0Xbj2A1t1zaPw1tYD6uBvJDbpuEc5fyJ/dYEvBL6bxKUXpIjOBgQylxka/EEQr8KZd3O5+lCD23zPsBamXvpOWLO511EdhjtUte2CVBLda/rvwAAAP//UEsHCKBYdcU/AQAA7AEAAFBLAQIUABQACAAIAAAAAACgWHXFPwEAAOwBAAAIAAAAAAAAAAAAAAAAAAAAAABvcmlnaW5hbFBLBQYAAAAAAQABADYAAAB1AQAAAAA=
    mountPath: /kafka-logs
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: rancher.io/local-path
    volume.kubernetes.io/selected-node: kafka3.solarfs.k8s
  finalizers:
  - kubernetes.io/pvc-protection
  generateName: kafka-2-storage-0-
  labels:
    app: kafka
    brokerId: "2"
    kafka_cr: kafka
  name: kafka-2-storage-0-9t4zd
  namespace: kafka
  ownerReferences:
  - apiVersion: kafka.banzaicloud.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: KafkaCluster
    name: kafka
    uid: f6dd87a3-405f-413c-a8da-6ff6b16105b9
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: local-path
  volumeMode: Filesystem
  volumeName: pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f
```

```shell
kubectl apply -f kafka2.new.pvc.yaml
```

查看pvc 是否正常绑定 对应pv

```shell
kubectl get pvc kafka-2-storage-0-9dl5s

# 显示如下
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
kafka-2-storage-0-9dl5s   Bound    pvc-c525fa5d-896a-4f88-b9f5-6e5c0726a8e1   10Gi       RWO            local-path     20mf
```

kafka3 主机备份目录移动为正式目录，并重置目录权限, kafka 

```shell
mv /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd.bak /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd
chmod 777 /data/local-path-provisioner/pvc-21082bbf-22f3-4ce3-8e73-9d4bd4ad2b8f_kafka_kafka-2-storage-0-9t4zd
```

#### 修改helm 安装values.yaml,放开注释broker id 2 的内容

```yaml
  brokers:
    - id: 0
      brokerConfigGroup: "default"
      brokerConfig:
        nodePortExternalIP:
          external: "172.16.152.20" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
    - id: 1
      brokerConfigGroup: "default"
      brokerConfig:
        nodePortExternalIP:
          external: "172.16.142.252" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
    - id: 2
      brokerConfigGroup: "default"
      brokerConfig:
        nodePortExternalIP:
         external: "172.16.233.132" # if "hostnameOverride" is not set for "external" external listener than broker is advertised on this IP
```

#### 更新kafka cluster 

```shell
kubectl apply -f kafkacluster-with-nodeport-external.yaml
```

### 迁移pod kafka-1 kafka-0 pv 数据

同pod kafka-2 pv 数据
