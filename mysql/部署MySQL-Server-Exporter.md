# 部署MySQL Server Exporter

## 数据库授权

```
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'localhost' IDENTIFIED BY "xxxxxx";
GRANT SELECT ON performance_schema.* TO 'exporter'@'localhost' IDENTIFIED BY "xxxxxx";

GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'%' IDENTIFIED BY "xxxxxx";
GRANT SELECT ON performance_schema.* TO 'exporter'@'%' IDENTIFIED BY "xxxxxx";

flush privileges;
```

## 发布服务yaml（包含service/ServiceMonitor）

mysqld_exporter.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mysqld-exporter
  name: mysqld-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysqld-exporter
      name: mysqld-exporter
  template:
    metadata:
      labels:
        app: mysqld-exporter
        name: mysqld-exporter
    spec:
      containers:
        - env:
            - name: DATA_SOURCE_NAME
              value: 'exporter:xxxxxx@(mysql.kont.svc.cluster.local:3306)/'
          image: 'prom/mysqld-exporter:v0.12.1'
          imagePullPolicy: IfNotPresent
          name: mysqld-exporter
          ports:
            - containerPort: 9104
              protocol: TCP          
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mysqld-exporter
  name: mysqld-exporter
  namespace: kont
  ports:
    - name: 9104-tcp
      port: 9104
      protocol: TCP
      targetPort: 9104
  selector:
    name: mysqld-exporter
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mysqld-exporter
  # Change this to the namespace the Prometheus instance is running in
  namespace: openshift-monitoring
  labels:
    k8s-app: mysqld-exporter
spec:
  selector:
    matchLabels:
      k8s-app: mysqld-exporter # target bfs-gateway service
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kont
  endpoints:
  - port: 9104-tcp
    scheme: http
    interval: 60s
```

## 导入grafana图表

下载下面图表json,导入到grafana

```
https://raw.githubusercontent.com/prometheus/mysqld_exporter/master/mysqld-mixin/dashboards/mysql-overview.json
```
