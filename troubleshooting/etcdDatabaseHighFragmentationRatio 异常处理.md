# etcdDatabaseHighFragmentationRatio 异常处理

## prometheus 中 告警中看到下面信息

```
name: etcdDatabaseHighFragmentationRatio
expr: (last_over_time(etcd_mvcc_db_total_size_in_use_in_bytes[5m]) / last_over_time(etcd_mvcc_db_total_size_in_bytes[5m])) < 0.5
for: 10m
labels:
   severity: warning
annotations:
   description: etcd cluster "{{ $labels.job }}": database size in use on instance {{ $labels.instance }} is {{ $value | humanizePercentage }} of the actual allocated disk space, please run defragmentation (e.g. etcdctl defrag) to retrieve the unused fragmented disk space.
   runbook_url: https://etcd.io/docs/v3.5/op-guide/maintenance/#defragmentation
   summary: etcd database size in use is less than 50% of the actual allocated storage.
```

## 解决方法

在master 主机执行下面命令

```
kubectl exec $(kubectl get pods --selector=component=etcd -A -o name \
| head -n 1) -n kube-system -- etcdctl defrag --cluster \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--key /etc/kubernetes/pki/etcd/server.key \
--cert /etc/kubernetes/pki/etcd/server.crt

```

## 参考

https://blog.alekc.org/posts/fixing-etcddatabasehighfragmentationratio-prometheus-alert/