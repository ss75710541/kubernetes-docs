## OpenFaaS定时任务

## 安装OpenFaas

[helm安装openfaas.md](./helm安装openfaas.md)

## 安装cron-connector

下载chart

```sh
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
helm pull openfaas/cron-connector
```

安装

```sh
helm upgrade --install \
cron-connector cron-connector-0.6.7.tgz \
    --namespace openfaas \
    --set image=registry.solarfs.io/openfaas/cron-connector:0.6.1
```



## 参考

https://artifacthub.io/packages/helm/openfaas/cron-connector

https://docs.openfaas.com/reference/cron/

https://zhuanlan.zhihu.com/p/632391041