# WalletConnect-Relay 部署

## 使用 quay.io 自动构建镜像

略

参考：

https://docs.quay.io/guides/building.html

https://access.redhat.com/documentation/zh-cn/red_hat_quay/3.6/html-single/use_red_hat_quay/index

## 编写helm chart

参考：

[快速编写通用helm chart](https://liujinye.gitbook.io/kubernetes-docs/cicd/kuai-su-bian-xie-tong-yong-helmchart)

https://github.com/paradeum-team/relay/blob/master/ops/package/docker-compose.override.yml

编写好的 helm chart 代码提交至 

https://github.com/paradeum-team/geth-helm-charts/tree/main/walletconnect-relay

## 使用argocd部署

### 部署WalletConnect-Relay 使用的 redis

dev-walletconnect-relay-redis.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-walletconnect-relay-redis
  namespace: cicd
spec:
  destination:
    name: k8s-test
    namespace: nft
  project: dev-nft
  source:
    chart: redis
    helm:
      parameters:
      - name: replica.replicaCount
        value: "0"
      - name: image.repository
        value: bitnami/redis
      - name: image.registry
        value: dockerproxy.com
      - name: auth.enabled
        value: "false"
      valueFiles:
      - values.yaml
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: 16.13.2
```

执行创建

```sh
kubectl apply -f dev-walletconnect-relay-redis.yaml
```



### 部署 WalletConnect-Relay

dev-walletconnect-relay.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-walletconnect-relay
  namespace: cicd
spec:
  destination:
    name: k8s-test
    namespace: nft
  project: dev-nft
  source:
    helm:
      parameters:
      - name: ingress.enabled
        value: "true"
      - name: ingress.className
        value: nginx
      - name: ingress.hosts[0].host
        value: dev-walletconnect-relay.example.com
      - name: redisUrl
        value: redis://dev-walletconnect-relay-redis-headless.nft.svc:6379/0
      valueFiles:
      - values.yaml
      values: |2-

        imagePullSecrets:
          - name: registry-pld-cicd

        ingress:
          tls:
            - secretName: example-com-tls
              hosts:
                - dev-walletconnect-relay.example.com
    path: walletconnect-relay
    repoURL: https://github.com/paradeum-team/geth-helm-charts.git
    targetRevision: dev
  syncPolicy:
    automated: {}
```

执行部署

```sh
kubectl apply -f dev-walletconnect-relay.yaml
```

## 参考

https://github.com/paradeum-team/relay

https://liujinye.gitbook.io/kubernetes-docs/cicd/argocd-tian-jia-fu-wu-liu-cheng