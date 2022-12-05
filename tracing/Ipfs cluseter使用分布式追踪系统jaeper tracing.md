# Ipfs cluseter使用分布式追踪系统jaeper tracing

## 依赖环境

- Kubernetes 1.19+
- Helm 3
- cert-manager 1.6.1+ 

## 安装jaeper operator

下载chart

```sh
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
helm pull jaegertracing/jaeger-operator
```

创建 values.yaml

```yaml
image:
  repository: registry.solarfs.io/jaegertracing/jaeger-operator
```

安装 jaeger-operator

```sh
helm upgrade --install jaeger-operator jaeger-operator-2.37.0.tgz -f values.yaml -n jaeger-system --create-namespace
```

## 安装jaeper

下载chart

```sh
helm pull jaegertracing/jaeger
```

创建 jaeger-es-elastic-user secret

```yaml
apiVersion: v1
data:
  elastic: xxxxxxxxxxxxxxxx
kind: Secret
metadata:
  name: jaeger-es-elastic-user
  namespace: jaeger-system
type: Opaque
```

```sh
kubectl apply -f jaeger-es-elastic-user.yaml
```

创建values.yaml

```yaml
provisionDataStore:
  cassandra: false
  elasticsearch: false
  kafka: false

storage:
  # allowed values (cassandra, elasticsearch)
  type: elasticsearch
  elasticsearch:
    scheme: http
    host: uat-es-internal-http.elastic-system.svc
    port: 9200
    user: elastic
    usePassword: true
    #password: changeme
    # indexPrefix: test
    ## Use existing secret (ignores previous password)
    existingSecret: jaeger-es-elastic-user
    existingSecretKey: elastic
    nodesWanOnly: false
    extraEnv: []
    ## ES related env vars to be configured on the concerned components
      # - name: ES_SERVER_URLS
      #   value: http://elasticsearch-master:9200
      # - name: ES_USERNAME
      #   value: elastic
      # - name: ES_INDEX_PREFIX
      #   value: test
    ## ES related cmd line opts to be configured on the concerned components
    cmdlineParams: {}
      # es.server-urls: http://elasticsearch-master:9200
      # es.username: elastic
      # es.index-prefix: test
  kafka:
    brokers:
      - kafka-0.kafka-headless.kafka.svc:9092
    topic: jaeger_v1_dev
    authentication: none

ingester:
  enabled: true
  image: registry.solarfs.io/jaegertracing/jaeger-ingester
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: prometheus-community

agent:
  image: registry.solarfs.io/jaegertracing/jaeger-agent
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: prometheus-community
  tolerations:
  - effect: NoSchedule
    key: ipfs-cluster-extend/cluster
    operator: Exists
  useHostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet

collector:
  image: registry.solarfs.io/jaegertracing/jaeger-collector
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: prometheus-community
  ingress:
    enabled: true
    ingressClassName: nginx
    tls:
       - hosts:
          - "jaeger-collector.apps145205.example.k8s"
    hosts: ["jaeger-collector.apps145205.example.k8s"]
    pathType: ImplementationSpecific

query:
  enabled: true
  extraEnv:
    # monitor
    - name: METRICS_STORAGE_TYPE
      value: "prometheus"
    - name: PROMETHEUS_SERVER_URL
      value: "http://prometheus-operated.monitoring:9090"
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: prometheus-community
  image: registry.solarfs.io/jaegertracing/jaeger-query
  ingress:
    enabled: true
    ingressClassName: nginx
    tls:
       - hosts:
          - "jaeger-query.apps145205.example.k8s"
    hosts: ["jaeger-query.apps145205.example.k8s"]
    pathType: ImplementationSpecific

esIndexCleaner:
  enabled: true
  image: registry.solarfs.io/jaegertracing/jaeger-es-index-cleaner

esRollover:
  enabled: true
  image: registry.solarfs.io/jaegertracing/jaeger-es-rollover

esLookback:
  enabled: true
  image: registry.solarfs.io/jaegertracing/jaeger-es-rollover

# 测试示例服务
hotrod:
  enabled: false
  podSecurityContext: {}
  securityContext: {}
  replicaCount: 1
  image:
    repository: registry.solarfs.io/jaegertracing/example-hotrod
  ingress:
    enabled: true
    ingressClassName: nginx
    tls:
       - hosts:
          - "jaeger-hotrod.apps145205.example.k8s"
    hosts: ["jaeger-hotrod.apps145205.example.k8s"]
    pathType: ImplementationSpecific
```

安装 jaeper

```sh
helm upgrade --install jaeger jaeger-0.64.2.tgz -n jaeger-system -f values.yaml
```

## 故障排查

### Monitor 数据为空



## 参考

https://github.com/jaegertracing/jaeger

https://github.com/xjjdog/example-jaeger-tracing

https://pjw.io/articles/2018/05/18/jaeger-tutorial/

https://github.com/jaegertracing/helm-charts

https://www.jaegertracing.io/docs/1.39/operator/

https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger-operator

https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger

https://www.jaegertracing.io/docs/1.39/spm/

https://www.jaegertracing.io/docs/1.39/spm/#query-prometheus

