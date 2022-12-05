# cert-manager-1.5升级到1.6

## 升级 cert-manager CRD 和 cert-manager 自定义资源的存储版本

在 1.4 版弃用后，不再提供cert-manager API 版本 v1alpha2、v1alpha3 和 v1beta1。

这意味着如果您的部署清单包含任何这些 API 版本，您将无法在升级后部署它们。我们新的 cmctl 实用程序或旧的 kubectl cert-manager 插件可以为您将旧清单转换为 v1。

## 现在按照常规升级过程

### 下载helm chart

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update jetstack
helm pull jetstack/cert-manager --version 1.6.3
```

如果你已经单独安装了CRD(而不是在你的Helm安装命令中添加`--set installCRDs=true`选项)，你应该先升级你的CRD资源:

```sh
wget -O cert-manager-v1.6.3.crds.yaml https://github.com/cert-manager/cert-manager/releases/download/v1.6.3/cert-manager.crds.yaml
kubectl apply -f cert-manager-v1.6.3.crds.yaml
```

执行helm 更新操作

```sh
helm upgrade --install  cert-manager  cert-manager-v1.6.3.tgz -n cert-manager --create-namespace \
--set image.repository=registry.hisun.netwarps.com/jetstack/cert-manager-controller \
--set webhook.image.repository=registry.hisun.netwarps.com/jetstack/cert-manager-webhook \
--set cainjector.image.repository=registry.hisun.netwarps.com/jetstack/cert-manager-cainjector \
--set startupapicheck.image.repository=registry.hisun.netwarps.com/jetstack/cert-manager-ctl
```

## 下载安装 cmctl 

```sh
wget https://github.com/cert-manager/cert-manager/releases/download/v1.6.3/cmctl-linux-amd64.tar.gz
tar xzvf cmctl-linux-amd64.tar.gz
mv cmctl /usr/local/bin/
ln -s /usr/local/bin/cmctl /usr/bin/cmctl
```

## 参考

https://cert-manager.io/docs/installation/upgrading/upgrading-1.5-1.6

https://cert-manager.io/docs/installation/upgrading/remove-deprecated-apis/#upgrading-existing-cert-manager-resources

https://cert-manager.io/docs/reference/cmctl/#migrate-api-version

https://cert-manager.io/docs/installation/upgrading/

