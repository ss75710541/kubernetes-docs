# cluster-monitor-operator添加外部metrics


默认prometheus-k8s 没有读取除openshift-monitoring 外的 project 权限 ，如果metrics 是在其它project中，需要添加sa prometheus-k8s 读取整个集群namespace 权限,执行下面操作：

```
oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:openshift-monitoring:prometheus-k8s
```

添加被监控服务的 svc 和 ServiceMonitor

注意下面yaml 中的 lebels 及 selector 的对应关系，否则会找不到

bfs-data-detection.yaml

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: bfs-data-detection
  name: bfs-data-detection
  namespace: bfs-webs
spec:
  ports:
  - name: http
    port: 9103
    protocol: TCP
    targetPort: http
  selector:
    app: bfs-data-detection
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bfs-data-detection
  # Change this to the namespace the Prometheus instance is running in
  namespace: openshift-monitoring
  labels:
    k8s-app: bfs-data-detection
spec:
  selector:
    matchLabels:
      k8s-app: bfs-data-detection # target bfs-data-detection service
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - bfs-webs
  endpoints:
  - port: http
    scheme: http
    interval: 5m
```

```
oc apply -f bfs-data-detection.yaml
```