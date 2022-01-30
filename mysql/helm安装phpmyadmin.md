# helm安装phpmyadmin

## 下载chart

```bash
 helm repo add bitnami https://charts.bitnami.com/bitnami
```

## 创建 values.yaml

```yaml
ingress:
  enabled: true
  hostname: testphpmyadmin.example.com
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  extraTls:
    - hosts:
        - testphpmyadmin.example.com
      secretName: example-com-tls
db:
  host: mysql.example.svc
```

## 执行安装

```bash
helm upgrade --install example-mysql-admin bitnami/phpmyadmin -n kugga-audio -f values.yaml
```

## 参考

https://artifacthub.io/packages/helm/bitnami/phpmyadmin