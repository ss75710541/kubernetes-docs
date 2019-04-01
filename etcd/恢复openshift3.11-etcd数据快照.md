# 恢复openshift3.11-etcd数据快照

参考：https://docs.openshift.com/container-platform/3.11/admin_guide/assembly_restoring-cluster.html#restoring-etcd-v2-v3-data

因openshift数据是以etcd v3版本存储，本恢复方法只编写了etcd v3版本的数据恢复，v2版本数据恢复不支持。

1.通过删除etcd pod定义并重新启动主机来停止所有etcd服务

```
mkdir -p /etc/origin/node/pods-stopped
mv /etc/origin/node/pods/* /etc/origin/node/pods-stopped/
```

2.所有etcd主机，清除所有旧数据，因为etcdctl将在将要执行还原过程的节点中重新创建它(这里为安全起见，使用mv把目录改名)

```
mv /var/lib/etcd /var/lib/etcd.old
```

3.所有etcd主机，运行snapshot restore命令：

```
. /etc/etcd/etcd.conf
etcdctl3 snapshot restore /backup/etcd-xxxxxx/db \
  --data-dir /var/lib/etcd \
  --name $ETCD_NAME \
  --initial-cluster "$ETCD_INITIAL_CLUSTER" \
  --initial-cluster-token "$ETCD_INITIAL_CLUSTER_TOKEN" \
  --initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS \
  --skip-hash-check=true
```

4.所有etcd主机将权限和selinux上下文还原到还原的文件：

```
chown -R etcd.etcd /var/lib/etcd/
restorecon -Rv /var/lib/etcd
```

5.所有etcd主机启动服务

```
mv /etc/origin/node/pods-stopped/* /etc/origin/node/pods/
```
6.所有etcd主机检查是否有错误日志

```
master-logs etcd etcd
```
7.检查openshift服务

```
oc get user
oc get all
```


## 支持作者

如果文章对您有帮助，欢迎打赏，谢谢

支付宝
![支付宝](../shoukuan.png)
