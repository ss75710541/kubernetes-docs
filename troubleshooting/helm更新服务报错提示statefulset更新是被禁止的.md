# helm更新服务报错提示statefulset更新是被禁止的

## helm更新服务时报错

```
helm upgrade --install dev-geth geth -f dev-values.yaml -n geth
```

```
invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'template', and 'updateStrategy' are forbidden
```

## 排查

查看helm 服务的历史版本

```
helm history dev-geth
```

显示如下：

```
2       	Fri Nov 26 18:25:49 2021	superseded	geth-0.2.1	1.10.1     	Upgrade complete
3       	Mon Dec  6 10:57:00 2021	failed    	geth-0.2.1	1.10.1     	Upgrade "dev-geth" failed: cannot patch "dev-geth" with kind StatefulSet: StatefulSet.apps "dev-geth" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'template', and 'updateStrategy' are forbidden
```

获取helm 更新前后历史版本更新前后的`manifest`,并对比更新

```
helm get manifest dev-geth --revision=2 > manifest.2.yaml
helm get manifest dev-geth --revision=3 > manifest.3.yaml
diff -c manifest.2.yaml manifest.3.yaml
```

显示如下：

```
*** manifest.2.yaml	2021-12-06 14:20:32.535072169 +0800
--- manifest.3.yaml	2021-12-06 14:20:35.276179626 +0800
***************
*** 6,13 ****
--- 6,15 ----
    name: dev-geth
    labels:
      helm.sh/chart: geth-0.2.1
+     geth/cluster: dev
      app.kubernetes.io/name: geth
      app.kubernetes.io/instance: dev-geth
+     geth/cluster: dev
      app.kubernetes.io/version: "1.10.1"
      app.kubernetes.io/managed-by: Helm
  ---
***************
*** 65,72 ****
--- 67,76 ----
    name: dev-geth
    labels:
      helm.sh/chart: geth-0.2.1
+     geth/cluster: dev
      app.kubernetes.io/name: geth
      app.kubernetes.io/instance: dev-geth
+     geth/cluster: dev
      app.kubernetes.io/version: "1.10.1"
      app.kubernetes.io/managed-by: Helm
  spec:
***************
*** 87,92 ****
--- 91,97 ----
    selector:
      app.kubernetes.io/name: geth
      app.kubernetes.io/instance: dev-geth
+     geth/cluster: dev
  ---
  # Source: geth/templates/statefulset.yaml
  apiVersion: apps/v1
***************
*** 95,102 ****
--- 100,109 ----
    name: dev-geth
    labels:
      helm.sh/chart: geth-0.2.1
+     geth/cluster: dev
      app.kubernetes.io/name: geth
      app.kubernetes.io/instance: dev-geth
+     geth/cluster: dev
      app.kubernetes.io/version: "1.10.1"
      app.kubernetes.io/managed-by: Helm
  spec:
***************
*** 105,110 ****
--- 112,118 ----
      matchLabels:
        app.kubernetes.io/name: geth
        app.kubernetes.io/instance: dev-geth
+       geth/cluster: dev
    serviceName: dev-geth
    template:
      metadata:
***************
*** 113,118 ****
--- 121,127 ----
        labels:
          app.kubernetes.io/name: geth
          app.kubernetes.io/instance: dev-geth
+         geth/cluster: dev
      spec:
        serviceAccountName: dev-geth
        securityContext:
***************
*** 147,152 ****
--- 156,162 ----
              - --config
              - /root/config/config.custom.toml
              - --http
+             - --ws
              - --allow-insecure-unlock
              - --rpc.allow-unprotected-txs
              - --gcmode=archive
```

Statefulset更新字段是受限的，禁止更新 'replicas', 'template', 'updateStrategy' 以外的字段 ，所以导致更新失败

## 解决

删除sts 重建，删除sts 默认不会删除pvc , 所以重建后业务也不会受影响

```shell
kubectl delete sts dev-geth
```

```
helm upgrade --install dev-geth geth -f dev-values.yaml -n geth
```

## 参考

https://github.com/helm/helm/issues/7998

https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets