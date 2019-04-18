# 在openshift使用3scale-operator部署3scale

## 制作3scale-operator镜像

注：hub.docker.com 中已经上传镜像，如果不需要更新镜像中内容，可以直接使用

```
docker pull docker.io/ss75710541/3scale-operator:latest
```

### 下载源码

```
mkdir -p $GOPATH/src/github.com/ss75710541
cd $GOPATH/src/github.com/ss75710541
git clone https://github.com/ss75710541/3scale-operator
cd 3scale-operator
git checkout master
```
### 更新镜像列表信息（不更新则忽略此步骤）

```
# 在下面源码文件中修改镜像信息
pkg/3scale/amp/component/ampimages.go
pkg/3scale/amp/product/release_2_5.go
pkg/3scale/amp/product/upstream.go
```

提交代码(注意：一定要提交代码，因为源码中引用的是github.com的全路径，下载到vendor中，所以代码制作发布模板会使用旧的版本)

```
git commit -am "update images"
git push
```

查看最后的commit id

```
git log|head -1
```

修改依赖 `vi Gopkg.toml`

编辑dep依赖文件 `Gopkg.toml`

```
[[override]]
  name = "github.com/ss75710541/3scale-operator"
  revision = "<填写上面查到的commit id>"
```

### 更新vendor

```
make vendor
```

### 重新生成发布相关服务的模板

```
cd pkg/3scale/amp/
make all
```

提交模板更新

```
git commit -am "update amp templates"
```

### 制作镜像并推送

```
cd $GOPATH/src/github.com/ss75710541/3scale-operator
export VERSION=v0.2.0.8
make build
make push
```

## 发布3scale-operator

```sh
# 以OpenShift管理员用户创建3scale-operator的CRDs:
for i in `ls deploy/crds/*_crd.yaml`; do oc create -f $i ; done

# 创建一个新的空项目(这可以用任何想要的OpenShift用户来完成)
export NAMESPACE="3scale"
oc new-project $NAMESPACE
oc project $NAMESPACE

# 创建 3scale-operator ServiceAccount
oc create -f deploy/service_account.yaml

# 创建 3scale-operator 相关的 roles和 role bindings 
oc create -f deploy/role.yaml
oc create -f deploy/role_binding.yaml

# 修改部署3cale-operator yaml中的镜像地址
# 最新的是latest 
sed -i 's|REPLACE_IMAGE|quay.io/3scale/3scale-operator:latest|g' deploy/operator.yaml

# 部署 3scale-operator
oc create -f deploy/operator.yaml

# 查看部署状态
oc get deployment 3scale-operator
```

## 部署APIManager自定义资源

```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  productVersion: <productVersion>
  wildcardDomain: <wildcardDomain>
  wildcardPolicy: <None|Subdomain>
  resourceRequirementsEnabled: true
```

示例：

```
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: example-apimanager
spec:
  productVersion: 2.5
  wildcardDomain: apigateway.hisun.com
  wildcardPolicy: None
  resourceRequirementsEnabled: true
```

如果wildcardPolicy是Subdomain，需要启用OpenShift路由器级别的通配符路由。可以通过执行下面命令来实现
`oc set env dc/router ROUTER_ALLOW_WILDCARD_ROUTES=true -n default`

## 相关参考文档

* [User guide](doc/user-guide.md)
* [APIManager reference](doc/apimanager-reference.md)
* [Tenant reference](doc/tenant-reference.md)
* [Capabilities reference](doc/api-crd-reference.md) 


## 支持作者

如果文章对您有帮助，欢迎打赏，谢谢

![支付宝](../shoukuan.png)
