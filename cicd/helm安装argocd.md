# helm安装argocd

## 下载chart

```
helm repo add argo https://argoproj.github.io/argo-helm
helm pull argo/argo-cd
```

## 编写values.yaml

```yaml
redis-ha:
  enabled: true

controller:
  enableStatefulSet: true

server:
  replicas: 2
  env:
    - name: ARGOCD_API_SERVER_REPLICAS
      value: '2'
  ingress:
    enabled: false
    https: false
    annotations:
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    ingressClassName: nginx
    hosts:
      - argocd-server.example.com
    tls:
      - hosts:
          - argocd-server.example.com
        secretName: example-com-tls
    certificate:
      enabled: false

repoServer:
  replicas: 2
  config:
    # Argo CD's externally facing base URL (optional). Required when configuring SSO
    url: https://argocd-server.example.com
    # Argo CD instance label key
    application.instanceLabelKey: argocd.argoproj.io/instance
		# keycloak sso
    oidc.config: |
      name: Keycloak
      issuer: https://sso.example.com/auth/realms/oc
      clientID: argocd
      clientSecret: $oidc.keycloak.clientSecret
      requestedScopes: ["openid", "profile", "email", "groups"]
```

## 执行安装

```shell
helm upgrade --install argo argo-cd-3.29.4.tgz -f values.yaml -n cicd --create-namespace
```

## 安装argocd cli

```
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

## 配置keycloak sso

### keycloak 添加client ，配置ArgoCD用户组

略

参考：https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak/

### 编辑argocd-cm

```shell
kubectl -n cicd edit cm argocd-cm
```

如下所示：

```yaml
apiVersion: v1
data:
  application.instanceLabelKey: argocd.argoproj.io/instance
  oidc.config: |
    name: Keycloak
    issuer: https://sso.paradeum.com/auth/realms/oc
    clientID: argocd
    clientSecret: $oidc.keycloak.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
  url: https://argocd-server-apps92250.solarfs.io
kind: ConfigMap
```

**注意：如果使用 helm upgrade 更新argocd 服务， 需要重新修改 argocd-cm**

```
kubectl -n cicd edit cm argocd-cm
```

### 配置ArgoCD权限策略

```sh
kubectl edit cm -n cicd argocd-rbac-cm
```

keycloak中 `/ArgoCDAdmins` 组用户默认admin角色，如下所示：

```
apiVersion: v1
data:
  policy.csv: |
    g, /ArgoCDAdmins, role:admin
kind: ConfigMap
```

**注意：组名ArgoCDAdmins前面必须有/, 否则不能匹配**

## 添加集群

### 使用argocd cli 登录 argocd

```sh
# 获取默认的admin 密码，登录
PASSWORD=$(kubectl get secret -n cicd argocd-secret -o jsonpath="{.data.admin\.password}"|base64 -d)
argocd login argo-argocd-server.cicd.svc --username admin --password "$PASSWORD"
```

### 添加rancher 集群

rancher 管理的K8s 集群，不能直接使用argocd  cli添加到clusters, argocd clusters 实际是与k8s crd 交互 ，所以我们手动添加k8s secret ，会自动添加到 argocd clusters 中

#### 创建cicd用户并创建 apikey，指定集群范围

略

#### 编写k8s-test-argocd-secret，apikey 显示值填写到下面yaml 内容中

```
apiVersion: v1
kind: Secret
metadata:
  namespace: cicd
  name: k8s-test-argocd-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: k8s-test
  server: "https://rancher.cattle-system.svc/k8s/clusters/<集群id>"
  config: |
    {
      "bearerToken": "<apikey 中显示的 bearerToken 信息>",
      "tlsClientConfig": {
        "insecure": true
      }
    }
```

```
kubectl apply -f k8s-test-argocd-secret.yaml
```

## 使用argocd cli 查看clusters 列表

```sh
argocd cluster list
```

显示如下

```sh
SERVER                                                       NAME             VERSION  STATUS      MESSAGE                                              PROJECT
https://rancher.cattle-system.svc/k8s/clusters/c-m-xxxxxxxx  k8s-test  1.22     Successful
https://kubernetes.default.svc                               in-cluster                Unknown     Cluster has no application and not being monitored.
```

## 添加 gitlab webhook

Argo CD每三分钟轮询一次Git存储库，以检测清单的更改。为了消除轮询的延迟，API服务器可以被配置为接收webhook事件。Argo CD支持GitHub, GitLab, Bitbucket, Bitbucket Server和Gogs的Git webhook通知。

### 添加webhook secret

#### 编辑 argocd-secret

```sh
kubectl edit secret -n cicd argocd-secret
```

在 data 中添加 webhook.github.secret，值需要使用base64编码

```
data:
  ...
  webhook.gitlab.secret: xxxxx
```

#### helm chart 项目添加webhook

**Chart 项目 --> `设置`->`集成` **

URL 输入

```
https://argocd-server.example.com/api/webhook
```

Secret Token 输入

```
<argocd-secret 中 webhook.gitlab.secret值base64解码>
```

选择`Trigger` 

 `Tag push events` 、`Push events`

点击 `Add webhhok`

## 参考

https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd

https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak/

https://docs.openshift.com/container-platform/4.7/cicd/gitops/configuring-sso-for-argo-cd-on-openshift.html

https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/

https://www.cnblogs.com/novwind/p/15358792.html

https://gist.github.com/janeczku/b16154194f7f03f772645303af8e9f80

