# helm+kube-prometheus-stack-prometheus-operator+local-path(storageclass)部署的prometheus迁移主机

## 主机规划

| 主机名   | ip            | node label   | 备注                   |
| -------- | ------------- | ------------ | ---------------------- |
| logging1 | 172.16.13.77  | logging=true | 原prometheus0 pvc 主机 |
| logging2 | 172.16.36.25  | logging=true | 原prometheus1 pvc 主机 |
| monitor1 | 172.16.54.227 | monitor=true | 新prometheus0 pvc 主机 |
| monitor2 | 172.16.84.64  | monitor=true | 新prometheus1 pvc 主机 |

## 备份prometheus

### 停止prometheus

```shell
kubectl patch prometheus prometheus-community-kube-prometheus --type="merge" -p '{"spec":{"replicas":0}}'
```

### 查看pvc

```shell
kubectl get pvc|grep prometheus-db

# 显示如下
prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0           Bound    pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e   50Gi       RWO            local-path     52d
prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1           Bound    pvc-2d48398a-91d8-47f5-973b-d03f37f0a898   50Gi       RWO            local-path     52d
```

### 备份pvc pv yaml

```shell
kubectl get pvc prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0 -o yaml > prometheos0.pvc.yaml
kubectl get pvc prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1 -o yaml > prometheus1.pvc.yaml
kubectl get pv pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e -o yaml > prometheus0.pv.yaml
kubectl get pv pvc-2d48398a-91d8-47f5-973b-d03f37f0a898 -o yaml > prometheus1.pv.yaml
```

### 备份pv数据

查看pv `prometheus0.pv.yaml` `prometheus1.pv.yaml`得知pv 所在主机及目录分别为:

```
logging1
```

```
/data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0
```

```
logging2
```

```
/data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1
```

在monitor1主机执行

```shell
# copy prometheos-0 pv 数据到monitor1临时目录
scp -r 172.16.13.77:/data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0 /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0.bak
```

monitor2 执行

```shell
# copy prometheos-1 pv 数据到monitor1临时目录
scp -r 172.16.36.25:/data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1 /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1.bak
```

## 迁移prometheus

### 设置monitor 主机label

```shell
kubectl label node monitor1 monitor=true
kubectl label node monitor2 monitor=true
```

### 删除旧pvc

```shell
kubectl delete pvc prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0 prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1
```

### 重建pv pvc

#### 重建prometheus-0 pv pvc 

复制备份pv 为新pv yaml

```shell
cp prometheus0.pv.yaml prometheus0.new.pv.yaml
```

编辑 prometheus0.new.pv.yaml，清理多余内容(主要是uid、resourceVersion、creationTimestamp、status等数据)， 并修改storage 大小，内容如下:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 200Gi
  hostPath:
    path: /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - monitor1.solarfs.k8s
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  volumeMode: Filesystem
```

创建新prometheus-0 pv

```shell
kubectl apply -f prometheus0.new.pv.yaml
```

复制备份pvc 到 prometheus0.new.pvc.yaml

```shell
cp prometheus0.pvc.yaml prometheus0.new.pvc.yaml
```

编辑prometheus0.new.pvc.yaml，清理多余内容(主要是uid、resourceVersion、creationTimestamp、status等数据)， 并修改storage 大小，内容如下:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: rancher.io/local-path
    volume.kubernetes.io/selected-node: monitor1.solarfs.k8s
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: prometheus
    app.kubernetes.io/instance: prometheus-community-kube-prometheus
    app.kubernetes.io/managed-by: prometheus-operator
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/version: 2.26.0
    operator.prometheus.io/name: prometheus-community-kube-prometheus
    operator.prometheus.io/shard: "0"
    prometheus: prometheus-community-kube-prometheus
  name: prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: local-path
  volumeMode: Filesystem
  volumeName: pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e
```

创建新prometheus-0 pvc

```shell
kubectl apply -f prometheus0.new.pvc.yaml
```

查看pvc 状态, Bound 状态为正常

```shell
# kubectl get pvc|grep prometheus-db
# 显示如下
prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0           Bound    pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e   200Gi      RWO            local-path     3h6m
```

#### 重建prometheus-1 pv pvc

步骤同prometheus-0, 略

#### 备份数据修改为pv path目录

在monitor1主机执行

```shell
mv /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0.bak /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0
# 重置目录权限及属主
chmod 777 /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0
chmod 777 /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0/prometheus-db
chown 1000:2000 -R  /data/local-path-provisioner/pvc-bde439cc-2889-4a81-ba68-f38a93c83c4e_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-0/prometheus-db/*
```

在monitro2 主机执行

```shell
mv /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1.bak /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1
# 重置目录权限及属主
chmod 777 /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1
chmod 777 /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1/prometheus-db
chown 1000:2000 -R /data/local-path-provisioner/pvc-2d48398a-91d8-47f5-973b-d03f37f0a898_monitoring_prometheus-prometheus-community-kube-prometheus-db-prometheus-prometheus-community-kube-prometheus-1/prometheus-db/*
```

### 使用helm 更新prometheus

#### 修改helm 更新values

kube-prometheus-stack-values.yaml, 增加 nodeSelector 配置

```yaml
prometheus:
  ...
  prometheusSpec:
    replicas: 2
    nodeSelector:
      monitor: 'true'
```

执行`helm upgrade`

```shell
helm upgrade --install prometheus-community kube-prometheus-stack-19.1.0.tgz -f kube-prometheus-stack-values.yaml -n  monitoring --create-namespace
```

检查prometheus pod

```shell
kubectl get pod|grep kube-prometheus
# 显示如下
prometheus-prometheus-community-kube-prometheus-0          2/2     Running   0          176m
prometheus-prometheus-community-kube-prometheus-1          2/2     Running   0          176m
```

