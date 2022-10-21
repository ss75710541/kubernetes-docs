# 使用argocd部署mongo-express

## 编辑dev-mongo-express.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-mongo-express
  namespace: cicd
spec:
  destination:
    name: k8s-example
    namespace: dev-example
  project: dev-example
  source:
    chart: mongo-express
    helm:
      parameters:
      - name: ingress.hosts[0].host
        value: dev-mongo-express.example.k8s
      - name: ingress.enabled
        value: "true"
      - name: ingress.ingressClassName
        value: nginx
      - name: mongodbServer
        value: dev-mongodb.dev-example.svc
      - name: basicAuthPassword
        value: xxxxxxxxxxxxx
      - name: basicAuthUsername
        value: root
      - name: mongodbAdminPassword
        value: xxxxxxxxxxxxxx
      - name: mongodbEnableAdmin
        value: "true"
      valueFiles:
      - values.yaml
      values: |-
        ingress:
          tls:
            - secretName: example-com-tls
              hosts:
                - dev-mongo-express.example.k8s
    repoURL: https://cowboysysop.github.io/charts/
    targetRevision: 2.7.3
  syncPolicy:
    automated: {}
```

## 部署

```sh
kubectl apply -f dev-mongo-express.yaml
```

## 参考

https://artifacthub.io/packages/helm/cowboysysop/mongo-express