#  使用helm+kube-prometheus-stack只部署prometheus

## 下载chart

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack
ls
```

## 创建kube-prometheus-stack-values.yaml

```yaml
defaultRules:
  create: false
grafana:
  enabled: false
alertmanager:
  enabled: false
prometheusOperator:
  enabled: false
kubeApiServer:
  enabled: false
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false


prometheus:
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      #cert-manager.io/cluster-issuer: selfsigned-issuer
    tls:
       - hosts:
          - "kong-prometheus.apps.example.k8s"
         #secretName: prometheus-tls
    hosts: ["kong-prometheus.apps.example.k8s"]
    pathType: ImplementationSpecific
  prometheusSpec:
    replicas: 2
    retention: 365d
    nodeSelector:
      node-role.kubernetes.io/kong-prometheus-chainstorage: 'true'
    image:
      repository: quay.io/prometheus/prometheus
      tag: v2.39.1
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    alerting:
      alertmanagers:
        - static_configs:
            - targets: ['prometheus-community-kube-alertmanager.monitoring:9093']
          scheme: http
          timeout: 5s
      alert_relabel_configs:
        - source_labels: [__name__]
          regex: '^(.*)$'
          replacement: '$1'
          target_label: alertname
```

##  安装

```sh
helm upgrade --install kong-prometheus-chainstorage kube-prometheus-stack-41.9.0.tgz -f kube-prometheus-stack-values.yaml -n  monitoring --create-namespace
```

## 参考

https://liujinye.gitbook.io/kubernetes-docs/prometheus-operator/k8s%201.22%20%E7%8E%AF%E5%A2%83%20kube-prometheus-stack%2022.x%20%E5%8D%87%E7%BA%A7%E8%87%B3%2041.x