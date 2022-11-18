# 使用image-syncer同步所需镜像到仓库

## 使用image-syncer同步所需镜像到仓库

### 安装 image-syncer

参考：https://github.com/AliyunContainerService/image-syncer/blob/master/README-zh_CN.md

### 编写 monitoring.yaml

```yaml
# kube-prometheus-stack
# https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml
k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.3.0: registry.solarfs.io/ingress-nginx/kube-webhook-certgen:v1.3.0
quay.io/prometheus-operator/prometheus-operator:v0.60.1: registry.solarfs.io/prometheus-operator/prometheus-operator:v0.60.1
quay.io/prometheus/alertmanager:v0.24.0: registry.solarfs.io/prometheus/alertmanager:v0.24.0
quay.io/prometheus/prometheus:v2.39.1: registry.solarfs.io/prometheus/prometheus:v2.39.1
quay.io/prometheus-operator/prometheus-config-reloader:v0.60.1: registry.solarfs.io/prometheus-operator/prometheus-config-reloader:v0.60.1
# kube-state-metrics
# https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-state-metrics/values.yaml
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.6.0: registry.solarfs.io/kube-state-metrics/kube-state-metrics:v2.6.0
# prometheus-node-exporter
# https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-node-exporter/values.yaml
quay.io/prometheus/node-exporter:v1.4.0: registry.solarfs.io/prometheus/node-exporter:v1.4.0
# grafana
# https://github.com/grafana/helm-charts/blob/main/charts/grafana/values.yaml
grafana/grafana:9.2.5: registry.solarfs.io/grafana/grafana:9.2.5
curlimages/curl:7.85.0: registry.solarfs.io/curlimages/curl:7.85.0
quay.io/kiwigrid/k8s-sidecar:1.19.2: registry.solarfs.io/kiwigrid/k8s-sidecar:1.19.2
busybox:1.31.1: registry.solarfs.io/library/busybox:1.31.1
```

### 编写auth.yaml

```yaml
registry.soalrfs.io:
  username: "cicd"
  password: "xxxxxxxxxx"
```

### 执行同步镜像列表

```sh
./image-syncer --auth auth.yaml --images monitoring.yaml --arch "amd64" --os linux
```