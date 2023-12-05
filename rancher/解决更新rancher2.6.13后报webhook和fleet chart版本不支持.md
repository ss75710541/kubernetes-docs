# 解决更新rancher2.6.13后报webhook和fleet chart版本不支持

## 部署方式

模板渲染

```sh
helm template rancher ./rancher-2.6.13.tgz --output-dir . \
--namespace cattle-system \
--set hostname=rancher.example.com \
--set replicas=2 \
--set ingress.tls.source=secret \
--set useBundledSystemChart=true \
-f values.yaml
```

执行更新

```
kubectl -n cattle-system apply -R -f ./rancher
```

## 报错信息

```
2023/12/05 08:35:33 [ERROR] available chart version (100.0.2+up0.3.8) for fleet is less than the min version (100.2.3+up0.5.3) 
2023/12/05 08:35:33 [ERROR] Failed to find system chart fleet will try again in 5 seconds: no chart name found
```

查看clusterrepos，发现这个版本 commit dc9ad74ba365f4ea15d173aac999f4e8134925f9 比较旧，不是最新的

```yaml
# kubectl get clusterrepos.catalog.cattle.io rancher-charts -o yaml

apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  creationTimestamp: "2021-12-31T04:06:55Z"
  generation: 4
  name: rancher-charts
  resourceVersion: "1199671909"
  uid: 86962934-4176-477d-ad0a-d8c8c0af7469
spec:
  forceUpdate: "2022-10-19T10:49:54Z"
  gitBranch: release-v2.6
  gitRepo: https://git.rancher.io/charts
status:
  branch: release-v2.6
  commit: 5d21c199dc7db29b6a5c755558edf4f6343b4c2b
  conditions:
  - lastUpdateTime: "2022-10-19T10:49:55Z"
    status: "True"
    type: FollowerDownloaded
  - lastUpdateTime: "2023-12-05T09:16:34Z"
    status: "True"
    type: Downloaded
  downloadTime: "2023-12-05T09:16:34Z"
  indexConfigMapName: rancher-charts-0-86962934-4176-477d-ad0a-d8c8c0af7469
  indexConfigMapNamespace: cattle-system
  observedGeneration: 4
  url: https://git.rancher.io/charts
```



## 解决方法

删除 rancher 自动创建的 clusterrepos

```
  kubectl delete clusterrepos.catalog.cattle.io rancher-charts
  kubectl delete clusterrepos.catalog.cattle.io rancher-rke2-charts
  kubectl delete clusterrepos.catalog.cattle.io rancher-partner-charts
```

重启rancher

```sh
kubectl rollout restart deploy rancher -n cattle-system
```

查看 clusterrepo rancher-charts, 发现已经更新

```yaml
# kubectl get clusterrepos.catalog.cattle.io rancher-charts -o yaml

apiVersion: catalog.cattle.io/v1
kind: ClusterRepo
metadata:
  creationTimestamp: "2023-12-05T09:28:51Z"
  generation: 1
  name: rancher-charts
  resourceVersion: "1199693562"
  uid: 5d24e500-2a6f-4b68-bd31-f85928ca1d54
spec:
  gitBranch: release-v2.6
  gitRepo: https://git.rancher.io/charts
status:
  branch: release-v2.6
  commit: dc9ad74ba365f4ea15d173aac999f4e8134925f9
  conditions:
  - lastUpdateTime: "2023-12-05T09:28:51Z"
    status: "True"
    type: FollowerDownloaded
  - lastUpdateTime: "2023-12-05T09:34:05Z"
    status: "True"
    type: Downloaded
  downloadTime: "2023-12-05T09:34:05Z"
  indexConfigMapName: rancher-charts-0-5d24e500-2a6f-4b68-bd31-f85928ca1d54
  indexConfigMapNamespace: cattle-system
  observedGeneration: 1
  url: https://git.rancher.io/charts
```

查看rancher 日志，也不再报 ` fleet is less than the min version` 相关错误



## 参考

https://github.com/harvester/harvester/blob/a9006087711e92415960801ceca611febd04e937/package/upgrade/upgrade_manifests.sh#L193-L199

https://github.com/rancher/rancher/issues/36914