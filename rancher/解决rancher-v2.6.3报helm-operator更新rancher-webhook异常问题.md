# 解决rancher-v2.6.3报helm-operator更新rancher-webhook异常问题

## 问题描述

`namespace` `Cattle-system`中 频繁创建pod`helm-operation-xxxxx`, 并且状态为Error

## 排查解决

### 查看helm-operator相关pod

```
kubectl get pod -n cattle-system

NAME                               READY   STATUS      RESTARTS       AGE
helm-operation-2mchw               1/2     Error       0              32m
helm-operation-54bq4               1/2     Error       0              22m
helm-operation-5zv78               1/2     Error       0              32m
helm-operation-6tbkg               1/2     Error       0              32m
helm-operation-7z6jn               1/2     Error       0              12m
helm-operation-9tps9               1/2     Error       0              37m
```

### 查看日志

```
kubectl logs helm-operation-9tps9 -n cattle-system -c helm

# 显示如下
helm upgrade --history-max=5 --install=true --namespace=cattle-system --reset-values=true --timeout=5m0s --values=/home/shell/helm/values-rancher-webhook-1.0.2-up0.2.2.yaml --version=1.0.2+up0.2.2 --wait=true rancher-webhook /home/shell/helm/rancher-webhook-1.0.2-up0.2.2.tgz
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

根据日志得知 helm 更新`rancher-webhook` 时, 有已经正在 安装/更新/回滚 的 进程

### 尝试查看helm 列表，没有看到已经部署的app

```
helm ls -n cattle-system

NAME           	NAMESPACE    	REVISION	UPDATED                               	STATUS         	CHART  
```

不显示`pending-install` 状态服务

### helm 增加 -a 参数后显示

```
helm ls -a -n cattle-system

NAME           	NAMESPACE    	REVISION	UPDATED                               	STATUS         	CHART                        	APP VERSION
rancher-webhook	cattle-system	1       	2021-12-31 04:07:38.73169718 +0000 UTC	pending-install	rancher-webhook-1.0.2+up0.2.2	0.2.2
```

### 卸载部署异常的helm app rancher-webhook 

```
helm uninstall rancher-webhook -n cattle-system
```

### 等待helm-operation-xxxxx 自动再次触发，并查看

```

[root@master1 rancher]# kubectl get pod -n cattle-system
NAME                           READY   STATUS              RESTARTS   AGE
helm-operation-ghlwj           1/2     Error               0          5m
helm-operation-h5rfs           1/2     Error               0          44m
helm-operation-hv4ht           1/2     Error               0          39m
helm-operation-jch7c           0/2     ContainerCreating   0          0s
```

查看日志为正常

```
kubectl logs helm-operation-jch7c -n cattle-system -c helm

# 显示如下
helm upgrade --history-max=5 --install=true --namespace=cattle-system --reset-values=true --timeout=5m0s --values=/home/shell/helm/values-rancher-webhook-1.0.2-up0.2.2.yaml --version=1.0.2+up0.2.2 --wait=true rancher-webhook /home/shell/helm/rancher-webhook-1.0.2-up0.2.2.tgz
Release "rancher-webhook" does not exist. Installing it now.
creating 6 resource(s)
beginning wait for 6 resources with timeout of 5m0s
Deployment is not ready: cattle-system/rancher-webhook. 0 out of 1 expected pods are ready
Deployment is not ready: cattle-system/rancher-webhook. 0 out of 1 expected pods are ready
Deployment is not ready: cattle-system/rancher-webhook. 0 out of 1 expected pods are ready
Deployment is not ready: cattle-system/rancher-webhook. 0 out of 1 expected pods are ready
Deployment is not ready: cattle-system/rancher-webhook. 0 out of 1 expected pods are ready
NAME: rancher-webhook
LAST DEPLOYED: Thu Oct 13 10:11:09 2022
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

---------------------------------------------------------------------
SUCCESS: helm upgrade --history-max=5 --install=true --namespace=cattle-system --reset-values=true --timeout=5m0s --values=/home/shell/helm/values-rancher-webhook-1.0.2-up0.2.2.yaml --version=1.0.2+up0.2.2 --wait=true rancher-webhook /home/shell/helm/rancher-webhook-1.0.2-up0.2.2.tgz
---------------------------------------------------------------------
```

查看pod 状态为正常

```
kubectl get pod helm-operation-jch7c -n cattle-system

NAME                   READY   STATUS      RESTARTS   AGE
helm-operation-jch7c   0/2     Completed   0          2m32s
```



## 参考

https://github.com/fluxcd/helm-controller/issues/149

https://forums.rancher.com/t/rancher-in-docker-helm-operation-error/37787