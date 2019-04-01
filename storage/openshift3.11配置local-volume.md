# openshift3.11配置local volume

参考：

https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner

https://docs.okd.io/3.11/install_config/configuring_local.html

https://ieevee.com/tech/2019/01/17/local-volume.html

## 下载local storage provisioner代码

```
git clone https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner.git
```

```
cd sig-storage-local-static-provisioner
```

## 生成local-volume-provisioner部署文件

如果不使用helm，或不方便安装heml，则跳过生成步骤，直接修改使用已经生成好的示例yaml配置，在`provisioner/deployment/kubernetes/example/`目录下

### 安装helm

mac安装helm

```
brew install kubernetes-helm
```


### 修改变量

vi helm/provisioner/values.yaml

注意：namespaces必须为kube-system,否则 local-volume-provisioner ds部署文件中的`priorityClassName: system-node-critical` 没有权限

```
...
  namespace: kube-system
...
  image: quay.io/external_storage/local-volume-provisioner:v2.3.0
...
prometheus:
  operator:
    enabled: true

    serviceMonitor:
      interval: 10s

      namespace: openshift-monitoring

      selector:
        k8s-app: local-volume-provisioner
```

### 生成部署文件

```
helm template ./helm/provisioner > ./provisioner/deployment/kubernetes/provisioner_generated.yaml
```

## 部署local-volume-provisioner

上传`provisioner/deployment/kubernetes/example/default_example_storageclass.yaml`到master1主机,可以酌情修改storageclass的名字，执行创建

```
oc create -f default_example_storageclass.yaml
```

上传provisioner_generated.yaml到master1主机，执行部署

```
kubectl create -f provisioner_generated.yaml
```

异常清理创建的相关服务（异常重建测试时使用）
  
```
oc delete -n kube-system ds local-volume-provisioner
oc delete -n kube-system configmap/local-provisioner-config
oc delete -n kube-system daemonset.apps/local-volume-provisioner
oc delete -n kube-system service/local-volume-provisioner
oc delete -n kube-system serviceaccount/local-storage-admin
oc delete -n openshift-monitoring servicemonitors local-volume-provisioner
oc delete -n kube-system sa local-storage-admin
oc delete -n kube-system clusterrolebinding local-storage-provisioner-pv-binding
oc delete -n kube-system clusterrole local-storage-provisioner-node-clusterrole
oc delete -n kube-system clusterrolebinding local-storage-provisioner-node-binding
```

## 创建测试挂载卷
  
```
mkdir /mnt/fast-disks
for vol in vol1 vol2 vol3; do
    mkdir /mnt/fast-disks/$vol
    mount -t tmpfs $vol /mnt/fast-disks/$vol
done
```


## 批量创建localvolume使用的卷

添加磁盘设备

创建pv

```
pvcreate /dev/vdb
```

创建vg

```
vgcreate vg_localvolume /dev/vdb
```

创建lvm

```
for i in {1..100};do
  mkdir /mnt/fast-disks/lv$i
  lvcreate -L 11G -n lv$i vg_localvolume
  mkfs.xfs /dev/mapper/vg_localvolume-lv$i
  echo "/dev/mapper/vg_localvolume-lv$i /mnt/fast-disks/lv$i xfs defaults 0 0" >> /etc/fstab
done
```

确认并挂载 

```
cat /etc/fstab
mount -a
```

清理lvm

```
for i in {1..100};do
  umount /dev/mapper/vg_localvolume-lv$i
  lvremove -y /dev/mapper/vg_localvolume-lv$i
  rm -r /mnt/fast-disks/lv$i
done

vgremove vg_localvolume
pvremove /dev/vdb
```

## 发布服务测试

statefulset-nginx-slim.yaml

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: example
  namespace: jyliu
spec:
  serviceName: nginx
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: nginx
          image: 'docker.io/googlecontainer/nginx-slim:0.8'
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: fast-disks
        resources:
          requests:
            storage: 1Gi

```

```
oc create -f statefulset-nginx-slim.yaml
```

## 支持作者

如果文章对您有帮助，欢迎打赏，谢谢

支付宝

![支付宝](../shoukuan.png)
