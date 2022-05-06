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

Dashboard 添加使用更简单一些，熟练之后可以直接使用yaml 添加，需要填写的内容如下:

**注意**：

argocd 配置中 Application 配置中

​	 helm.values 为 yaml 配置

​	 helm.parameters 配置是 直接修改单个参数

​	 helm.values 和 helm.parameters 都不修改表示使用 helm chart 中默认值

​	 需要修改的参数只选择一个地方修改，防止配置两个地方容易混淆出错, 如果不确定配置的参数是否生效，可以去已经是同步成功状态的 `svc/deploy/ing` 中查看配置项是否已经达到预期

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
      valueFiles:
      - values.yaml
      values: |-
        imagePullSecrets:
          - name: registry-pld-cicd
        env:
          - name: SERVER_PORT
            value: "8188"
          - name: MONGODB_HOSTS
            value: dev-app-mongodb.dev-app.svc:27017
          - name: MONGODB_DBNAME
            value: "foundation"
          - name: DAPP_HOST
            value: "http://dev-app-api.dev-app.svc:8188"

        ingress:
          enabled: true
          className: "nginx"
          hosts:
            - host: dev-app.example.com
              paths:
                - path: /
                  pathType: ImplementationSpecific
          tls:
            - secretName: example-com-tls
              hosts:
                - dev-app-sync-index.example.com
        extraVolumeMounts:
          - mountPath: /data/logs
            name: volume-logs
          - mountPath: /etc/localtime
            name: localtime
            readOnly: true
        extraVolumes:
          - name: localtime
            hostPath:
              path: /etc/localtime
          - name: volume-logs
            hostPath:
              path: /data/dev-app/logs
              type: ''
    path: charts/app
    repoURL: https://gitlab.example.com/pld/app-helm-charts.git
    targetRevision: dev
  syncPolicy:
    automated: {}
```

