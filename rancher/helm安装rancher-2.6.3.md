# helm 安装rancher 2.6.3

## 添加repo, 下载chart

```sh
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm update
helm fetch rancher-stable/rancher
```

## 创建namespace

```sh
kubectl create ns cattle-system
```

## 创建tls secret

```sh
kubectl create secret tls example.io-tls --cert=_.example.io.pem --key=_.example.io.key -n cattle-system
```

## 创建 values.yaml

```yaml
ingress:
  extraAnnotations:
    kubernetes.io/ingress.class: "nginx"
  tls:
    # options: rancher, letsEncrypt, secret
    source: secret
    secretName: solarfs.io-tls
```

## 生成rancher/template 目录

```sh
helm template rancher ./rancher-2.6.3.tgz --output-dir . \
--namespace cattle-system \
--set hostname=rancher.example.io \
--set replicas=2 \
--set useBundledSystemChart=true \
-f values.yaml

#--set systemDefaultRegistry=registry.cn-hangzhou.aliyuncs.com
```

## 安装rancher

```
kubectl -n cattle-system apply -R -f ./rancher
```

## 参考

https://github.com/ss75710541/operator-env/blob/main/rancher/helm%E7%BA%BF%E4%B8%8B%E5%AE%89%E8%A3%85rancher.md

https://nimtechnology.com/2021/08/25/argocd-add-other-cluster-rancher-or-k8s-into-argocd/

https://www.cnblogs.com/novwind/p/15358792.html

