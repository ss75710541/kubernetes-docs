# openshift3.11部署eck1.6+es7.14.1

## 部署eck-operator

```
wget https://download.elastic.co/downloads/eck/1.6.0/all-in-one.yaml
oc apply -f all-in-one.yaml
```

## 部署elasticsaerch

创建 es-elasticsearch.yaml

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es
  namespace: elastic-system
spec:
  version: 7.14.1
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  nodeSets:
  - name: master
    count: 1 # 数量
    config:
      node.roles: ["master", "data", "ingest", "ml", "transform"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # pvc 名称不支持修改
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 99Gi
        storageClassName: fast-disks # 当前使用的 storageclass 
    podTemplate:
      spec:
        nodeSelector:
          node-role.kubernetes.io/logging: 'true'
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          securityContext:
            privileged: true
```

```
oc apply -f es-elasticsearch.yaml
```

## 部署 Kibana

创建 es-kibana.yaml

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: es
  namespace: elastic-system
spec:
  version: 7.14.1
  count: 1
  elasticsearchRef:
    name: es
```

```
oc apply -f es-kibana.yaml
```

查看kibana登录密码
账号 elastic
```
kubectl get secret es-es-elastic-user -o=jsonpath='{.data.elastic}' -n elastic-system | base64 --decode; echo
```

## 创建elasticsaerch 过滤pipeline

在 `https://grokdebug.herokuapp.com/` 在线调试 Grok 正则表达式

通过 kibana dev-tools console 设置 grok 正则 pipeline

通过 kibana dev tools grokdebugger https://eck-kb.xxx.xxx/app/dev_tools#/grokdebugger 测试正则

打开 kibana 自带的 dev-tools console https://eck-kb.xx.xxx/app/dev_tools#/console 创建pipeline

```
PUT /_ingest/pipeline/access_pipeline
{
    "description":"access pipeline",
    "processors":[
        {
            "grok":{
                "field":"message",
                "patterns":[
"""^\[(?<app_name>[^ ]*)\] \[(?<hostname>[^ ]*)\] (?<ip>[^ ]*) - \[(?<logtime>.*)\] (?<method>[^ ]*) (?<uri>[^ ]*) (?<http_version>[^ ]*) (?<http_code>[^ ]*) (?<response_time>[^ ]*) (?<body_size>[^ ]*) (?<x_request_id>[^ ]*) "(?<other>[^"]*)"""
                ]
            }
        }
    ]
}
```

```
PUT /_ingest/pipeline/business_pipeline
{
    "description":"business pipeline",
    "processors":[
        {
            "grok":{
                "field":"message",
                "patterns":[
"""^\[(?<level>.*)\] (?<logtime>%{YEAR}-%{MONTHNUM}-%{MONTHDAY}[T ]%{HOUR}:?%{MINUTE}(?::?%{SECOND})) (?<message>.*)"""
                ]
            }
        }
    ]
}
```

## 部署 filebeat

创建 filebeat.yaml

```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: filebeat
  namespace: elastic-system
spec:
  type: filebeat
  version: 7.14.1
  elasticsearchRef:
    name: es
  config:
    filebeat.inputs:
    - type: log
      paths:
      - /aos/pn-api/logs/access_*.log
      fields:
        app: pn-api
        type: access
      pipeline: access_pipeline
    - type: log
      multiline.type: pattern
      multiline.pattern: '^\['
      multiline.negate: true
      multiline.match: after
      paths:
      - /aos/pn-api/logs/pn_*.log
      fields:
        app: pn-api
        type: business
      pipeline: business_pipeline
    - type: log
      paths:
      - /data/bfs-gateway/runtime/logs/access.*.log
      fields:
        app: gw
        type: access
      pipeline: access_pipeline
    - type: log
      multiline.type: pattern
      multiline.pattern: '^\['
      multiline.negate: true
      multiline.match: after
      paths:
      - /data/bfs-gateway/runtime/logs/gw.*.log
      fields:
        app: gw
        type: business
      pipeline: business_pipeline
  daemonSet:
    podTemplate:
      spec:
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true
        securityContext:
          runAsUser: 0
        containers:
        - name: filebeat
          securityContext:
            runAsUser: 0
            # If using Red Hat OpenShift uncomment this:
            privileged: true
          volumeMounts:
          - name: aos
            mountPath: /aos
            readOnly: true
          - name: data
            mountPath: /data
            readOnly: true
          - mountPath: /etc/localtime
            name: localtime
            readOnly: true
        volumes:
        - name: aos
          hostPath:
            path: /aos
        - name: data
          hostPath:
            path: /data
        - name: localtime
          hostPath:
            path: /etc/localtime
```

## 部署 journalbeat

参考： https://raw.githubusercontent.com/elastic/cloud-on-k8s/1.6/config/recipes/beats/journalbeat_hosts.yaml


创建 journalbeat.yaml

```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: journald
  namespace: elastic-system
spec:
  type: journalbeat
  version: 7.14.1
  elasticsearchRef:
    name: es # 关联的 es 名称
  config:
    journalbeat.inputs:
    - paths: []
      seek: cursor
      cursor_seek_fallback: tail
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
  daemonSet:
    podTemplate:
      spec:
        automountServiceAccountToken: true # some older Beat versions are depending on this settings presence in k8s context
        dnsPolicy: ClusterFirstWithHostNet
        containers:
        - name: journalbeat
          volumeMounts:
          - mountPath: /var/log/journal
            name: var-journal
          - mountPath: /run/log/journal
            name: run-journal
          - mountPath: /etc/machine-id
            name: machine-id
          securityContext:
            runAsUser: 0
            # If using Red Hat OpenShift uncomment this:
            privileged: true
        hostNetwork: true # Allows to provide richer host metadata
        securityContext:
          runAsUser: 0
        terminationGracePeriodSeconds: 30
        volumes:
        - hostPath:
            path: /var/log/journal
          name: var-journal
        - hostPath:
            path: /run/log/journal
          name: run-journal
        - hostPath:
            path: /etc/machine-id
          name: machine-id
```

```
oc apply -f  journalbeat.yaml
```