# Argocd添加服务流程

## 添加 Project

`Settings` -->`projects`--> `new project`

## 添加 Git repo

`Settings` -->`Repositories`--> `connect repo using https`

填写相应信息点击`connect`

## 编辑 Project

使用argocd cli 编辑 project

```sh
argocd proj edit dev-project
```

或在dashboard 页面操作 dev-project

相关内容如下

```yaml
clusterResourceWhitelist:
- group: '*'
  kind: '*'
destinations:
- name: ucloud-k8s-test
  namespace: nft
  server: https://rancher.cattle-system.svc/k8s/clusters/c-m-twqzhxkd
sourceRepos:
- https://gitlab.paradeum.com/pld/nft-helm-charts.git
```

## 添加 Application

可以通过命令行，添加Application , 也可以通过Dashboard 添加

Dashboard 添加使用更简单一些，熟练之后可以直接使用yaml 添加，需要填写的内容如下

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-app
  namespace: cicd
spec:
  destination:
    namespace: nft
    server: https://rancher.cattle-system.svc/k8s/clusters/c-m-xxxxxxxx
  project: dev-app
  source:
    helm:
      parameters:
      - name: env[4].value
        value: dev-mysql.nft.svc
      - name: envFrom[0].secretRef.name
        value: dev-app-secret
      - name: ingress.hosts[0].host
        value: dev-app.example.com
      - name: ingress.enabled
        value: "true"
      - name: ingress.className
        value: nginx
      valueFiles:
      - values.yaml
      values: |-
        imagePullSecrets:
          - name: registry-cicd
    path: charts/app
    repoURL: https://gitlab.example.com/project/app-helm-charts.git
    targetRevision: dev
  syncPolicy:
    automated: {}
```

