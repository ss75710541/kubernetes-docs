# k8s使用config-syncer(kubed)同步secret

## 集群内同步

### 使用argocd 部署 config-syncer(kubed)

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-config-syncer
  namespace: cicd
spec:
  destination:
    name: k8s-test
    namespace: kube-system
  project: dev-config-syncer
  source:
    chart: kubed
    helm:
      valueFiles:
      - values.yaml
    repoURL: https://charts.appscode.com/stable/
    targetRevision: v0.13.2
```

### 源 Secret 设置 `kubed.appscode.com/sync` Namespace Selector 注解

```sh
kubectl annotate secret netwarps-com-tls  kubed.appscode.com/sync="tls=netwarps" -n kube-system
kubectl annotate secret registry-pld-cicd  kubed.appscode.com/sync="registry=registry-pld-cicd" -n kube-system
```

### 同步目标namespace 设置对应label

```sh
for ns in adminer-system bigphoto dev-art-exhibition dev-cellar dev-foundation dev-whitetigger elastic-system geth ipfs nft tekton-pipelines test-art-exhibition test-cellar test-foundation  test-whitetigger
do
	kubectl label namespace $ns tls=netwarps
	kubectl label namespace $ns registry=registry-pld-cicd
done
```

### 查看同步的registry secret

```
kubectl get secret -A |grep registry-pld-cicd
adminer-system                           registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
bigphoto                                 registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      242d
dev-art-exhibition                       registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      29d
dev-cellar                               registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      32d
dev-foundation                           registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      179d
dev-whitetigger                          registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13d
elastic-system                           registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
geth                                     registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
ipfs                                     registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
kube-system                              registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      15m
nft                                      registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      34d
tekton-pipelines                         registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      31d
test-art-exhibition                      registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      29d
test-cellar                              registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      51d
test-foundation                          registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      177d
test-whitetigger                         registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13d
```

## 跨集群同步

### 使用argocd 部署 config-syncer(kubed)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-config-syncer
  namespace: cicd
spec:
  destination:
    name: cloud-k8s-test
    namespace: kube-system
  project: dev-config-syncer
  source:
    chart: kubed
    helm:
      valueFiles:
      - values.yaml
      values: |-
        config:
          kubeconfigContent: |-
            apiVersion: v1
            kind: Config
            clusters:
            - name: "cloud-k8s-test"
              cluster:
                server: "https://rancher.example.com/k8s/clusters/c-m-xxxxxxxx"
            # 不支持同一个域名同一个端口同步多个k8s(例如rancher 代理k8s apiserver情况下)
            #- name: "geth-uat-k8s"
            #  cluster:
            #    server: "https://rancher.example.com/k8s/clusters/c-m-xxxxxxxx"
            #- name: "geth-prod-k8s"
            #  cluster:
            #    server: "https://rancher.example.com/k8s/clusters/c-m-xxxxxxxx"

            users:
            - name: "cloud-k8s-test"
              user:
                token: "xxxxxxxx"
            - name: "geth-uat-k8s"
              user:
                token: "xxxxxxxx"
            - name: "geth-prod-k8s"
              user:
                token: "xxxxxxxx"


            contexts:
            - name: "cloud-k8s-test"
              context:
                user: "cloud-k8s-test"
                cluster: "cloud-k8s-test"
            - name: "geth-uat-k8s"
              context:
                user: "geth-uat-k8s"
                cluster: "geth-uat-k8s"
            - name: "geth-prod-k8s"
              context:
                user: "geth-prod-k8s"
                cluster: "geth-prod-k8s"

            current-context: "cloud-k8s-test"
    repoURL: https://charts.appscode.com/stable/
    targetRevision: v0.13.2
```



```sh
kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync-contexts="geth-uat-k8s,geth-prod-k8s" -n kube-system
kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync-contexts- -n kube-system
kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync="registry=registry-pld-cicd" -n kube-system


kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync-contexts="geth-uat-k8s,geth-prod-k8s" -n kube-system

kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync-contexts- -n kube-system
kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync- -n kube-system
kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync="tls=netwarps" -n kube-system
```

**注意: 不支持rancher 代理的多k8s集群，会误判为同一个集群地址，导致不能跨集群同步**



## 参考：

https://github.com/kubeops/config-syncer

https://appscode.com/products/kubed/v0.12.0/setup/install/

https://appscode.com/products/kubed/v0.12.0/

https://appscode.com/products/kubed/v0.12.0/guides/config-syncer/intra-cluster/