# 使用cronjob备份etcd

## 创建pvc

在kube-system项目中手动,选择storage class，创建名称`etcd-backup` 的pvc

## 发布备份cronjob 

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  namespace: kube-system
  name: etcd-backup
spec:
  schedule: "0 */12 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:          
            backup: "etcd"
        spec:
          containers:
          - name: etcd-backup
            image: quay.io/coreos/etcd:v3.2.22
            args:
              - |
                #!/bin/sh
                set -ex
                source /etc/etcd/etcd.conf
                mkdir /backup/etcd-$(date +%d%H)
                ETCDCTL_API=3 etcdctl --cert="/etc/etcd/peer.crt" --key=/etc/etcd/peer.key --cacert="/etc/etcd/ca.crt" --endpoints=$ETCD_ADVERTISE_CLIENT_URLS snapshot save /backup/etcd-$(date +%d%H)/db
                tar -zcvf /backup/etcd-$(date +%d%H).tar.gz /backup/etcd-$(date +%d%H)/db
                rm -Rf /backup/etcd-$(date +%d%H)
            command:
              - /bin/sh
              - '-c'
            securityContext:
              privileged: true
            volumeMounts:
              - mountPath: /etc/etcd/
                name: master-config
                readOnly: true
              - name: backup
                mountPath: /backup
          restartPolicy: OnFailure
          nodeSelector:
            node-role.kubernetes.io/master: 'true'
          volumes:
            - hostPath:
                path: /etc/etcd/
                type: ''
              name: master-config
            - name: backup
              persistentVolumeClaim:
                claimName: etcd-backup
```

## 检查备份数据

找到pvc etcd-backup 对应的网盘目录

```
oc get pvc|awk '{print $3}'|tail -1|xargs oc describe pv
```

```
Name:            pvc-7b9ad3fd-3bf5-11e9-84bc-00163e026866
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by=alicloud/nas
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    alicloud-nas
Status:          Bound
Claim:           kube-system/etcd-backup
Reclaim Policy:  Delete
Access Modes:    RWX
Capacity:        50Gi
Node Affinity:   <none>
Message:
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    12318f4b8e1-aot76.cn-hongkong.nas.aliyuncs.com
    Path:      /kube-system-etcd-backup-pvc-7b9ad3fd-3bf5-11e9-84bc-00163e026866
    ReadOnly:  false
Events:        <none>
```

挂载到临时目录

```
mount 12318f4b8e1-aot76.cn-hongkong.nas.aliyuncs.com:/kube-system-etcd-backup-pvc-7b9ad3fd-3bf5-11e9-84bc-00163e026866 /mnt/pvc
```

```
ll /mnt/pvc
total 84388
-rw-r--r--. 1 root root 12417398 Mar  1 15:57 etcd-0107.tar.gz
-rw-r--r--. 1 root root 12306442 Mar  1 20:00 etcd-0112.tar.gz
-rw-r--r--. 1 root root 12375795 Mar  2 08:00 etcd-0200.tar.gz
-rw-r--r--. 1 root root 12306346 Mar  2 20:00 etcd-0212.tar.gz
-rw-r--r--. 1 root root 12368600 Mar  3 08:00 etcd-0300.tar.gz
-rw-r--r--. 1 root root 12282545 Mar  3 20:00 etcd-0312.tar.gz
-rw-r--r--. 1 root root 12341568 Mar  4 08:00 etcd-0400.tar.gz
```

卸载目录

```
umount /mnt/pvc
```


## 支持作者

如果文章对您有帮助，欢迎打赏，谢谢

![支付宝](../shoukuan.png)
