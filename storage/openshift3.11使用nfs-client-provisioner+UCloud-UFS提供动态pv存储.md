# openshift3.11使用nfs-client-provisioner+UCloud-UFS提供动态pv存储

## 在UCloud创建 NFS 协议的UFS文件系统

创建UFS挂载点，挂载点地址为：172.16.16.137:/

准备部署使用的yaml

rbac.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

nfs-client-provisioner.yaml

其中NFS_SERVER 为 172.16.16.137, UFS 的  nfs接口不支持 nfs-client-provisioner 默认的挂载参数 ，所有需要手动 nfs-client-provisioner 使用的pvc和pv 配和nfs-client-provisioner发布

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      nodeSelector:
        node-role.kubernetes.io/infra: 'true'
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:v3.1.0-k8s1.11
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 172.16.16.137
            - name: NFS_PATH
              value: /
      volumes:
        - name: nfs-client-root
          persistentVolumeClaim:   #这里由volume改为persistentVolumeClaim
             claimName: nfs-client-root  
---
# 创建PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-client-root
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi  # 不必纠结是否与实际UFS 申请的存储大小一致，因为这个是不是后期storageclass创建的动态pv
---
# 创建pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-client-root
spec:
  capacity:
    storage: 200Gi  # 不必纠结是否与实际UFS 申请的存储大小一致，因为这个是不是后期storageclass创建的动态pv
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /
    server: 172.16.16.137  #这里直接写UFS的Server地址即可。
  mountOptions:
    - nolock
    - nfsvers=4.0
```

storageclass.yaml

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-nfs-storage
mountOptions:
  - nolock,tcp,noresvport
  - nfsvers=4.0
provisioner: fuseim.pri/ifs  # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

## 部署

```
oc create -f rbac.yaml
oc create -f nfs-client-provisioner.yaml
oc create -f storageclass.yaml
```

## 所有需要挂载nfs的宿主机开放selinux pod 挂载nfs权限

```
setsebool virt_use_nfs on
```

参考：

https://liujinye.gitbook.io/openshift-docs/storage/openshift3.11-shi-yong-nfsclientprovisioner+ali-yun-nas-ti-gong-dong-tai-nfs

https://docs.ucloud.cn/uk8s/volume/dynamic_ufs

