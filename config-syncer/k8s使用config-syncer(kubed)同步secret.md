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
    name: cloud-k8s-test
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

### 在kube-system 创建源 secret/configmap 配置

略

### 源 Secret 设置 `kubed.appscode.com/sync` Namespace Selector 注解

```sh
kubectl annotate secret netwarps-com-tls  kubed.appscode.com/sync="tls=netwarps" -n kube-system
kubectl annotate secret registry-pld-cicd  kubed.appscode.com/sync="registry=registry-pld-cicd" -n kube-system
```

### 同步目标namespace 设置对应label

```sh
for ns in adminer-system bigphoto elastic-system geth ipfs nft tekton-pipelines
do
	kubectl label namespace $ns tls=netwarps
	kubectl label namespace $ns registry=registry-pld-cicd
done
```

### 查看同步的registry secret

```
kubectl get secret -A |grep registry-pld-cicd
adminer-system                           registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
bigphoto                                 registry-pld-cicd                                                                                              kubernetes.io/dockerconfigjson        1      13d
elastic-system                           registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
geth                                     registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
ipfs                                     registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      13m
kube-system                              registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      15m
nft                                      registry-pld-cicd                                                kubernetes.io/dockerconfigjson        1      34d
tekton-pipelines                         registry-pld-cicd                                                
```

### 清理集群内同步的secret,谨慎操作，会同步删除集群内所有关联secret

```sh
# kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync- -n kube-system
# kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync- -n kube-system
```

## 跨集群同步

### 使用argocd 部署 config-syncer(kubed)

**注：默认的config-syncer 跨集群同步，不支持rancher 代理的k8s cluster api地址, 相同的域名端口会被误判为同一个集群，导致同步失败，下面部署用的config-syncer 镜像是我修改过代码支持了rancher 代理后重新制作的docker 镜像**

```
quay.io/netwarps/kubed@sha256:05e894200d732407763c552ce81633b79e63f7c6e7aaee5fcc7a13e1e1660b4c
```

完整argocd 部署 `prod-config-syncer.yaml` 如下:（部署到和rancher 相同k8s 集群,同步时使用rancher 内部代理后的k8s地址）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-config-syncer
  namespace: cicd
spec:
  destination:
    name: cloud-hk-kugga-prod
    namespace: kube-system
  project: prod-config-syncer
  source:
    chart: kubed
    helm:
      parameters:
      # 默认为unicorn，同步后会打入同步后对应配置的label
      - name: config.clusterName
        value: cloud-hk-kugga-prod
      - name: logLevel
        value: "5"
      - name: operator.tag
        value: 05e894200d732407763c552ce81633b79e63f7c6e7aaee5fcc7a13e1e1660b4c
      - name: operator.registry
        value: quay.io/netwarps
      - name: operator.repository
        value: kubed@sha256
      valueFiles:
      - values.yaml
      values: |-
        config:
          kubeconfigContent: |-
            apiVersion: v1
            kind: Config
            clusters:
            - name: "geth-uat-k8s"
              cluster:
                insecure-skip-tls-verify: true
                server: "https://rancher.cattle-system.svc/k8s/clusters/c-m-xxxxxxxx"
            - name: "geth-prod-k8s"
              cluster:
                insecure-skip-tls-verify: true
                server: "https://rancher.cattle-system.svc/k8s/clusters/c-m-xxxxxxxx"
            - name: "cloud-k8s-test"
              cluster:
                insecure-skip-tls-verify: true
                server: "https://rancher.cattle-system.svc/k8s/clusters/c-m-xxxxxxxx"

            users:
            - name: "geth-uat-k8s"
              user:
                token: "kubeconfig-u-gdwznkhv4p:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
            - name: "geth-prod-k8s"
              user:
                token: "kubeconfig-u-gdwzng6j9j:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
            - name: "cloud-k8s-test"
              user:
                token: "kubeconfig-u-v5rjddpgpx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"


            contexts:
            - name: "geth-uat-k8s"
              context:
                user: "geth-uat-k8s"
                cluster: "geth-uat-k8s"
            - name: "geth-prod-k8s"
              context:
                user: "geth-prod-k8s"
                cluster: "geth-prod-k8s"
            - name: "cloud-k8s-test"
              context:
                user: "cloud-k8s-test"
                cluster: "cloud-k8s-test"
    repoURL: https://charts.appscode.com/stable/
    targetRevision: v0.13.2
```

### 在kube-system 创建源 secret/configmap 配置

略

### 配置跨集群secret同步的集群（默认同步kube-system namespace）

```sh
kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync-contexts="cloud-k8s-test,geth-uat-k8s,geth-prod-k8s" -n kube-system
kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync-contexts="cloud-k8s-test,geth-uat-k8s,geth-prod-k8s" -n kube-system
```

**注：相关配置只同步到指定目标k8s 集群的 kube-system namespace ，如果需要同步其它namespace 参考集群内同步配置**

### 配置集群内同步的secret

```sh
kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync="tls=netwarps" -n kube-system
kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync="registry=registry-pld-cicd" -n kube-system
```

### 清理跨集群secret同步的集群，谨慎操作，会同步删除关联目标集群secret

```sh
# kubectl annotate secret netwarps-com-tls kubed.appscode.com/sync-contexts- -n kube-system
# kubectl annotate secret registry-pld-cicd kubed.appscode.com/sync-contexts- -n kube-system
```

## 参考：

https://github.com/kubeops/config-syncer

https://appscode.com/products/kubed/v0.12.0/setup/install/

https://appscode.com/products/kubed/v0.12.0/

https://appscode.com/products/kubed/v0.12.0/guides/config-syncer/intra-cluster/

https://github.com/paradeum-team/config-syncer/tree/jyliu-dev/charts/kubed