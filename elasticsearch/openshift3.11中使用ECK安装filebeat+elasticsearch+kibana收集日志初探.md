# openshift3.11中使用ECK安装filebeat+elasticsearch+kibana收集日志初探

## 部署eck-operator

```
# 部署eck-operator
kubuectl apply -f https://download.elastic.co/downloads/eck/1.3.1/all-in-one.yaml
```

## 部署elasticsaerch

其中`quickstart`可以修改为自定义的集群名称

elasticsearch.yaml

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.10.1
  nodeSets:
  - name: default
    count: 1
    config:
      node.master: true
      node.data: true
      node.ingest: true
    podTemplate:
      spec:
        nodeSelector:
          node-role.kubernetes.io/infra: 'true'
```

## 部署kibana

kibana.yaml

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.10.1
  count: 1
  elasticsearchRef:
    name: quickstart
  podTemplate:
    spec:
      nodeSelector:
        node-role.kubernetes.io/infra: 'true'
```

查看kibana登录密码

账号 elastic

```
kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

## 创建elasticsaerch 过滤pipline

### 方式一：通过curl 调用es api 设置 pipeline

bfs-api-pipeline.json

```
{"description":"bfs api pipeline","pipeline":{"processors":[{"grok":{"field":"message","patterns":["^\\[(?<app_name>[^\\]]*)\\] (?<ip>[^ ]*) - (?<logtime>[^\\]]*)\\] \\\"(?<method>[^ ]*) (?<url>[^ ]*) (?<http_version>[^ ]*) (?<http_code>[^ ]*) (?<response_time>[^ ]*)"]}}]},"docs":[{"_source":{"message":"[afs-PNode-api] x.x.x.x - [2020-12-16 14:44:00] \"GET /qn/sys/healthy HTTP/1.1 200 92.779µs \"go-resty/1.12.0 (https://github.com/go-resty/resty)\" \""}}]}
```

```
# 通过api 创建 elasticsaerch 使用gork正则匹配的pipline
curl -k -X PUT "http://quickstart-es-http.elastic-system.svc:9200/_ingest/pipeline/bfs_api_pipeline"  -H 'Content-Type: application/json' -d@bfs-api-pipeline.json
```

### 方式二：通过 kibana dev-tools console 设置 grok 正则 pipeline

通过 kibana dev tools grokdebugger `https://eck-kb.xxx.xxx/app/dev_tools#/grokdebugger` 测试正则

打开 kibana 自带的 dev-tools console `https://eck-kb.xx.xxx/app/dev_tools#/console` 创建pipeline

```
PUT /_ingest/pipeline/bfs_api_pipeline
{
    "description":"bfs api pipeline",
    "processors":[
        {
            "grok":{
                "field":"message",
                "patterns":[
"""^\[(?<app_name>[^\]]*)\] (?<ip>[^ ]*) - \[(?<logtime>[^\]]*)\] "(?<method>[^ ]*) (?<url>[^ ]*) (?<http_version>[^ ]*) (?<http_code>[^ ]*) (?<response_time>[^ ]*) "(?<other>[^"]*)"""
                ]
            }
        }
    ]
}
```

模拟数据测试pipeline

```
POST _ingest/pipeline/_simulate
{
    "description":"bfs api pipeline",
    "pipeline":{
        "processors":[
            {
                "grok":{
                    "field":"message",
                    "patterns":[
                        """^\[(?<app_name>[^\]]*)\] (?<ip>[^ ]*) - \[(?<logtime>[^\]]*)\] "(?<method>[^ ]*) (?<url>[^ ]*) (?<http_version>[^ ]*) (?<http_code>[^ ]*) (?<response_time>[^ ]*) "(?<other>[^"]*)"""
                    ]
                }
            }
        ]
    },
    "docs":[
        {
            "_source":{
                "message":"[afs-PNode-api] x.x.x.x - [2020-12-16 14:44:00] \"GET /qn/sys/healthy HTTP/1.1 200 92.779µs \"go-resty/1.12.0 (https://github.com/go-resty/resty)"}}]}
```


## 部署filebeat

filebeat.yaml

```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
spec:
  type: filebeat
  version: 7.10.1
  elasticsearchRef:
    name: quickstart
  config:
    filebeat.inputs:
    - type: log
      paths:
      - /aos/pn-api/logs/access_*.log
    output.elasticsearch:
      pipeline: 'bfs_api_pipeline'
  daemonSet:
    podTemplate:
      spec:
        nodeSelector:
          rnode: 'true'
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
          - mountPath: /etc/localtime
            name: localtime
            readOnly: true
        volumes:
        - name: aos
          hostPath:
            path: /aos
        - name: localtime
          hostPath:
            path: /etc/localtime
```

## 参考链接：

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html 

https://www.elastic.co/guide/en/elasticsearch/reference/master/grok-processor.html#grok-processor

https://cloud.tencent.com/developer/article/1643602

https://zhuanlan.zhihu.com/p/105453664