# helm部署adminer

## 添加helm repo

```sh
helm repo add paradeum https://paradeum-team.github.io/helm-charts
helm repo update
```

## 创建values.yaml

```yaml
service:
  type: ClusterIP

ingress:
  enabled: true
  ingressClassName: nginx
  hosts:
    - host: adminer.apps145205.pldtest.k8s
      paths:
      - path: /
        pathType: ImplementationSpecific
  tls:
    - secretName: netwarps-com-tls
      hosts:
        - adminer.apps145205.pldtest.k8s
```

## 安装

```sh
helm upgrade --install adminer adminer-0.1.9.tgz -n adminer-system --create-namespace -f values.yaml
```

