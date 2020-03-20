# Openshift3.11 alertmanager 持久化

## 初始安装配置为持久化

参考: https://docs.openshift.com/container-platform/3.11/install_config/prometheus_cluster_monitoring.html#monitoring-prerequisites

修改ansible 配置

```
openshift_cluster_monitoring_operator_alertmanager_storage_enabled=true
openshift_cluster_monitoring_operator_alertmanager_storage_class_name= custom-nas-storage
```

## 已经安装好的alertmanager修改为持久化

编辑`cluster-monitoring-config`，添加 `volumeClaimTemplate` 内容

```
oc edit configmap/cluster-monitoring-config -n openshift-monitoring
```

```
alertmanagerMain:
  baseImage: openshift/prometheus-alertmanager
  nodeSelector:
    node-role.kubernetes.io/infra: "true"
  hostport: "alertmanager-main-openshift-monitoring.apps181.hisun.com"
  volumeClaimTemplate:
    spec:
      storageClassName:  custom-nas-storage
      resources:
        requests:
          storage: 2Gi
```

## 异常处理

查看prometheus-operator 日志发现 ，更新alertmanager报错，提示禁止更新除'replicas'、 'template'、 和 'updateStrategy' 之外的其他字段的statefulset spec

```
E0320 03:55:44.263151       1 operator.go:278] Sync "openshift-monitoring/main" failed: updating statefulset failed: StatefulSet.apps "alertmanager-main" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'template', and 'updateStrategy' are forbidden.
```

因为不能修改，所以需要删除alertmanager statefulset 重建

```
oc delete sts/alertmanager-main -n openshift-monitoring
```

```
oc get sts/alertmanager-main -n openshift-monitoring -o yaml 
```

发现 alertmanager-main的yaml 配置已经存在 volumeClaimTemplates 使用storageclass 的相关配置
