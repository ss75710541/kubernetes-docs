# openshift3.11-pvc-delete-Terminating-hang


删除pvc 提示不成功，查看pvc pv 状态为 Terminating

```
# oc get pv pvc-7bd48605-d879-11e9-8e70-00163e0382f7
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                        STORAGECLASS   REASON    AGE
pvc-7bd48605-d879-11e9-8e70-00163e0382f7   1Gi        RWX            Delete           Terminating   bfs-webs/bos-platform-data   example-nfs              183d

# oc get pvc platform-data
NAME                    STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
platform-data       Terminating   pvc-7bd48605-d879-11e9-8e70-00163e0382f7   1Gi        RWX            example-nfs          183d
```

处理方法：

手动编写pvc pv, 删除 finalizers 所有内容

```
oc edit pvc xx
```

```
oc edit pv 
```

```
finalizers
  - kubernetes.io/pv-protection
```
