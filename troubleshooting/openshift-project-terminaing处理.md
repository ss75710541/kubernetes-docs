# openshift project Terminaing处理

参考：https://access.redhat.com/solutions/3881901

```
oc delete project kubevirt
```
卡住

```
# oc get project kubevirt
NAME                       DISPLAY NAME   STATUS
kubevirt                                  Terminating

```

显示 残留的有可能异常的 finalizer

```
# oc api-resources --verbs=list --namespaced -o name | xargs -n 1 oc get --show-kind --ignore-not-found -o jsonpath='{range .items[*]}{.kind}/{.metadata.name}: {.metadata.finalizers}{"\n"}{end}'
KubeVirt/kubevirt: [foregroundDeleteKubeVirt]
```

如果在执行获取残留列表过程中有类似如下报错

```
error: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: the server is currently unable to handle the request
```

根据报错 APIs找到异常 apiservice 的全名，并删除

```
# kubectl get apiservice | grep 'metrics.k8s.io/v1beta1'

v1beta1.metrics.k8s.io	2020-08-29T15:05:38Z
# oc delete apiservice v1beta1.metrics.k8s.io
```


清理 finalizers

```
# oc patch kubevirt/kubevirt --type=merge -p '{"metadata": {"finalizers":null}}'
kubevirt.kubevirt.io/kubevirt patched
```
