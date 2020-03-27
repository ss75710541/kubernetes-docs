# openshift-3.11-kubevirt从v0.19.0升级到v0.27.0

参考：https://kubevirt.io/user-guide/#/installation/updating-and-deleting-installs

## 升级前检查各服务是否正常

```
oc get pod -n kube-system
oc get pod -n cdi
oc get pod -n kubevirt
oc get pod -n kubevirt-web-ui
```

异常则处理异常，正常情况下向下操作（必须，处理异常，否则以下更新会失败）

##  尝试使用第一种更新

这种方法升级小版本有效，跨大版本升级有可能因为virt-opertor 使用了新 RBAC规则而失败，所以升级大版本建议第二种方法更新

修改 kubevirt-config index.docker.io 替换为 dockerhub.azk8s.cn

```
oc edit cm kubevirt-config
```

```
kubectl patch kv kubevirt -n kubevirt --type=json -p '[{ "op": "add", "path": "/spec/imageTag", "value": "v0.27.0" }]'
```

## 尝试第二种更新方式

```
# 添加 kubevirt-operator 特权
oc adm policy add-scc-to-user privileged -n kubevirt -z kubevirt-operator
export RELEASE=v0.27.0
wget https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
# 修改为国内源
sed -i 's/index.docker.io/dockerhub.azk8s.cn/g' kubevirt-operator.yaml
# 跨版本升级有可能需要新的RBAC权限 ，所以删除旧的创建新的
oc delete ClusterRoleBinding kubevirt-operator 
# 跨版本升级有可能 virt-operator 部分字段不能修改，所以删除重建
oc delete deployment virt-operator
kubectl apply -f kubevirt-operator.yaml
```


## 限制 kubevrit virt-handler 发布主机

```
kubectl patch ds/virt-handler -n kubevirt -p '{"spec": {"template": {"spec": {"nodeSelector": {"vm": "true"}}}}}'
```

## 发布vm测试

```

apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: alpine310-v0270-virt
spec:
  dataVolumeTemplates:
    - metadata:
        name: alpine310-v0270-virt-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: custom-nas-storage
        source:
          http: 
            url: http://172.26.163.182:8081/packages/kubevirt-image/alpine310-virt-base-0.8.qcow2.gz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        vm.kubevirt.io/name: alpine310-v0270-virt
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: vm
                operator: In
                values:
                - 'true'
      domain:
        cpu:
          cores: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
          interfaces:
            - bootOrder: 2
              masquerade: {}
              name: nic0
          rng: {}
        machine:
          type: ''
        resources:
          requests:
            memory: 64M
      hostname: alpine310-v0270-virt
      networks:
        - name: nic0
          pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
        - dataVolume:
            name: alpine310-v0270-virt-rootdisk
          name: rootdisk
```

vm pod logs 报错

```
oc logs virt-launcher-example-497w7 -c compute
{"component":"virt-launcher","level":"info","msg":"Collected all requested hook sidecar sockets","pos":"manager.go:68","timestamp":"2020-03-25T01:33:39.856069Z"}
{"component":"virt-launcher","level":"info","msg":"Sorted all collected sidecar sockets per hook point based on their priority and name: map[]","pos":"manager.go:71","timestamp":"2020-03-25T01:33:39.856213Z"}
panic: chown /var/run/kubevirt-private/a10e65b1-6e38-11ea-8d40-00163e0382f7: operation not permitted

goroutine 1 [running]:
main.initializeDirs(0x7ffe227e1e0c, 0x11, 0x7ffe227e1e33, 0x21, 0x7ffe227e1e6a, 0x21, 0x7ffe227e1dc2, 0x24)
	cmd/virt-launcher/virt-launcher.go:189 +0x4d0
main.main()
	cmd/virt-launcher/virt-launcher.go:362 +0x73f
{"component":"virt-launcher","level":"error","msg":"dirty virt-launcher shutdown","pos":"virt-launcher.go:531","reason":"exit status 2","timestamp":"2020-03-25T01:33:39.862363Z"}
{"component":"virt-launcher","level":"error","msg":"Failed to reap process -1","pos":"virt-launcher.go:496","reason":"no child processes","timestamp":"2020-03-25T01:33:39.862442Z"}
```

v0.21.0 之后，kubevirt 关闭了 virt-launcher 容器特权 privileged=true，但openshift环境pod 会提示 `operation not permitted`, 所有需要在vm 发布主机禁用selinux

```
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/g'  /etc/sysconfig/selinux
```

禁用selinix 之后，vm 启动成功


清理Terminating 状态project

参考：https://github.com/kubernetes/kubernetes/issues/60807

```
TOKEN=xxxx

for item in `oc get project |grep Terminating|awk '{print $1}'`
do
	kubectl get namespace $item -o json|grep -v kubernetes > $item.json
	curl -k -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -X PUT --data-binary @${item}.json https://172.26.163.195:8443/api/v1/namespaces/$item/finalize
done
```

## kubevirt web ui

升级到 v2.0.1

```
kubectl patch KWebUI kubevirt-web-ui -n kubevirt-web-ui --type=json -p '[{ "op": "add", "path": "/spec/version", "value": "v2.0.1" }]'
```

kubevrit web ui 新版本已经弃用直接部署，建议使用openshift console 包括kubevirt 管理，所以quay.io 的目前最新版本就只是到v2.0.1