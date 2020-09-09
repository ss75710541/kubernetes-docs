# openshift3.11-cluster-monitoring-operator数据持久化


## 创建动态pv storageclass

参考：[openshift3.11使用nfs-client-provisioner+UCloud-UFS提供动态pv存储.md](../storage/openshift3.11使用nfs-client-provisioner+UCloud-UFS提供动态pv存储.md)

## 创建local volume storageclass

参考：[openshift3.11配置local-volume.md](../storage/openshift3.11配置local-volume.md)

## 准备local volume 磁盘

分别给 两台 infra 主机创建 prometheus 需要的本地挂载磁盘, 并执行下面操作

```
# 格式化
mkfs.xfs /dev/vdb
# 创建挂载目录
mkdir -p /mnt/fast-disks/prometheus-k8s-data
# 添加持久化挂载配置
echo '/dev/vdb	/mnt/fast-disks/prometheus-k8s-data	auto	defaults,nofail,discard,comment=cloudconfig	0	2' >> /etc/fstab
# 执行挂载命令
mount -a
```
挂载后 登录master1 主机,就可以看到 local volume 的pv了, 因为在某些云厂商添加的磁盘，计算单位进位是1000，所以在挂载后显示pv 是 199Gi.

```
# oc get pv

local-pv-334f56e5                          199Gi RWO  ...
local-pv-4aaf7cd8                          199Gi RWO  ...
``` 

## 配置 ConfigMap cluster-monitoring-config 

alertmanagerMain 使用nfs 动态 storage class: custom-nfs-storage

prometheusK8s 使用local volume storage class: fast-disks

修改完成配置后 cluster-monitoring-config，会自动修改promehteus-k8s 的yaml 并重启

修改完整yaml 内容如下：

```
apiVersion: v1
data:
  config.yaml: |
    prometheusOperator:
      baseImage: quay.io/coreos/prometheus-operator
      prometheusConfigReloaderBaseImage: quay.io/coreos/prometheus-config-reloader
      configReloaderBaseImage: quay.io/coreos/configmap-reload
      nodeSelector:
        node-role.kubernetes.io/infra: "true"
    prometheusK8s:
      baseImage: openshift/prometheus
      nodeSelector:
        node-role.kubernetes.io/infra: "true"
      externalLabels:
        cluster: master.offline-okd.com
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: fast-disks
          resources:
            requests:
              storage: 199Gi
    alertmanagerMain:
      baseImage: prometheus/alertmanager
      nodeSelector:
        node-role.kubernetes.io/infra: "true"
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: custom-nfs-storage
          resources:
            requests:
              storage: 20Gi
    nodeExporter:
      baseImage: prometheus/node-exporter
    grafana:
      baseImage: grafana/grafana
      nodeSelector:
        node-role.kubernetes.io/infra: "true"
    kubeStateMetrics:
      baseImage: offlineregistry.offline-okd.com:5000/coreos/kube-state-metrics
      nodeSelector:
        node-role.kubernetes.io/infra: "true"
    kubeRbacProxy:
      baseImage: offlineregistry.offline-okd.com:5000/coreos/kube-rbac-proxy
    auth:
      baseImage: openshift/oauth-proxy
kind: ConfigMap
metadata:
  namespace: openshift-monitoring
```