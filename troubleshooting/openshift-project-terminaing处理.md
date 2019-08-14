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

清理 finalizers

```
# oc patch kubevirt/kubevirt --type=merge -p '{"metadata": {"finalizers":null}}'
kubevirt.kubevirt.io/kubevirt patched
```
