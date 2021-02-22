# openshift3.11清理Terminating 状态project

参考：https://github.com/kubernetes/kubernetes/issues/60807

获取project json内容，删除 finalizers 中 kubernetes

```
  finalizers:
  - kubernetes
```

可以先尝试通过 kubectl edit namespace xxx， 删除finalizers，如果不生效，可以使用下面脚本通过api清理, 其中TOKEN 为管理用户的TOKEN

```
TOKEN=xxxx

for item in `kubectl get project |grep Terminating|awk '{print $1}'`
do
	kubectl get namespace $item -o json|grep -v '"kubernetes"' > $item.json
	curl -k -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X PUT --data-binary @${item}.json https://172.26.163.195:8443/api/v1/namespaces/$item/finalize
done
```
