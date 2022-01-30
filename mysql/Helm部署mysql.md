# Helm部署mysql

## 添加 helm chart 源，下载chart 包 

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm pull bitnami/mysql
```

## 创建values.yaml

```yaml
global:
  storageClass: local-path

image:
  registry: docker.io
  repository: bitnami/mysql
  tag: 5.7.37

primary.persistence.storageClass: local-path
secondary.persistence.storageClass: local-path
```

## 安装

```sh
helm upgrade --install --namespace nft dev-mysql mysql-8.8.17.tgz -f dev-values.yaml
```

## 参考

https://artifacthub.io/packages/helm/bitnami/mysql

https://github.com/bitnami/charts/tree/master/bitnami/mysql