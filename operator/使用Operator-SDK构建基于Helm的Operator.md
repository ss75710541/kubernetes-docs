# 使用Operator-SDK构建基于Helm 的 Operator 

## 步骤

#### 创建一个项目目录并初始化项目

```sh
# mkdir geth-operator
# cd geth-operator
# operator-sdk init --domain example.com --plugins helm
```

#### 根据本地helm chart 包创建简单api

```sh
# operator-sdk create api \
      --helm-chart=/Users/liujinye/github/helm-charts/geth-0.2.4.tgz
```

#### 构建operator镜像并push到仓库

```sh
make docker-build docker-push IMG="registry.example.com/geth/geth-helm-operator:v0.0.3"
```

#### 因默认的gcr.io 仓库国内不能访问，修改默认 `config/default/manager_auth_proxy_patch.yaml` 中 image 如下

```yaml
				...
        image: gcr.dockerproxy.com/kubebuilder/kube-rbac-proxy:v0.11.0
        ...
```

## OLM 部署

#### 安装 olm

```sh
operator-sdk olm install
```

#### bundle operator  

```sh
VERSION=0.0.3
make bundle IMAGE_TAG_BASE=registry.example.com/geth/geth-helm-operator VERSION=$VERSION
IMG=registry.example.com/geth/geth-helm-operator:v$VERSION
# 显示如下内容：
operator-sdk generate kustomize manifests -q

Display name for the operator (required):
> geth-helm-operator

Description for the operator (required):
> geth helm operator

Provider's name for the operator (required):
> liujinye

Any relevant URL for the provider name (optional):
>

Comma-separated list of keywords for your operator (required):
> geth, app, operator

Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required):
> 75710541@qq.com

# 显示如下
cd config/manager && /Users/liujinye/github/geth-helm-operator/bin/kustomize edit set image controller=registry.example.com/geth/geth-operator:v0.0.2
/Users/liujinye/github/geth-helm-operator/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1
INFO[0000] Creating bundle.Dockerfile
INFO[0000] Creating bundle/metadata/annotations.yaml
INFO[0000] Bundle metadata generated suceessfully
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully
```

#### 制作operator bundle镜像并推送到仓库

```sh
VERSION=0.0.3
make bundle-build bundle-push IMAGE_TAG_BASE=registry.example.com/geth/geth-helm-operator VERSION=$VERSION IMG=registry.example.com/geth/geth-helm-operator:v$VERSION
```

#### 使用operator bundle 部署operator

```sh
operator-sdk run bundle registry.example.com/geth/geth-helm-operator-bundle:v0.0.3
```

#### 创建示例 Geth 自定义资源

```sh
kubectl apply -f config/samples/charts_v1alpha1_geth.yaml
```

#### 卸载operator

```sh
operator-sdk cleanup geth-helm-operator
```

## 直接部署

#### 部署operator

```sh
make deploy IMG="registry.netwarps.com/geth/geth-helm-operator:v0.0.3"
```

#### 创建示例 Geth 自定义资源

```sh
kubectl apply -f config/samples/charts_v1alpha1_geth.yaml
```

### 卸载operator

```sh
make undeploy
```

## 参考

https://sdk.operatorframework.io/docs/building-operators/helm/

https://sdk.operatorframework.io/docs/installation/

https://kind.sigs.k8s.io/docs/user/configuration/

https://sdk.operatorframework.io/docs/olm-integration/tutorial-bundle/

https://sdk.operatorframework.io/docs/olm-integration/cli-overview/#private-bundle-and-catalog-image-registries