# 使用 Tekton 构建CI流程

## 安装 Tekton

### 安装 Tekton 核心及Tekton pipelines

```sh
curl -o tekton-pipeline.yaml https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### 允许同一个Pipeline挂载两个pvc

默认tekton 禁用了同一个pipeline 挂载两个不同pvc, 当前 TriggerTemplate 示例中 source、 cache 两个workspaces 使用了不同的pvc, 所以需要开放限制

编辑 tekton-pipeline.yaml，找到  configmap feature-flags， 看到 disable-affinity-assistant, 值修改为true 。

如果安装前没有修改，也可以安装后`kubectl edit configmap feature-flags -n tekton-pipelines` 修改

### 安装 tekton-pipeline

```
kubectl apply -f tekton-pipeline.yaml
```

#### 配置Tekton在运行工作流时使用 manual 存储类请求10Gi的持久化卷

```
kubectl create configmap config-artifact-pvc \
                         --from-literal=size=10Gi \
                         --from-literal=storageClassName=manual \
                         -o yaml -n tekton-pipelines \
                         --dry-run=client | kubectl replace -f -
```

检查pods

```sh
kubectl get pods --namespace tekton-pipelines
```

### 安装 Tekton Cli

```sh
# CentOS8
dnf copr enable chmouel/tektoncd-cli
dnf install tektoncd-cli
```

```sh
tkn -h
```

### 安装 Tekton Triggers

```bash
curl -o tekton-triggres.yaml https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename tekton-triggres.yaml
curl -o tekton-triggers-interceptors.yaml https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
kubectl apply --filename tekton-triggers-interceptors.yaml
```

检查 pods

```bash
kubectl get pods --namespace tekton-pipelines
```

### 安装 Tekton Dashboard

```bash
curl -o tekton-dashboard.yaml https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml
kubectl apply --filename tekton-dashboard.yaml
```

检查 pods

```sh
kubectl get pods --namespace tekton-pipelines
```

创建 tekton-dashboard-ingress.yaml, tekton 没有用户管理，所以建议限制ip段访问

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dashboard-ingress
  namespace: tekton-pipelines
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: 'true'
    #nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # 限制ip 访问，默认不限制
    nginx.ingress.kubernetes.io/whitelist-source-range: "172.17.0.0/16,127.0.0.1,10.0.0.0/24"
spec:
  tls:
    - hosts:
        - tekton.example.com
      secretName: example-com-tls
  rules:
  - host: tekton.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
```

## 构建 CI 流程

![Tekton工作流程图.png](./images/Tekton工作流程图.png)

### 概念描述

#### Tekton Pipelines 实体列表

| 实体          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `Task`        | 定义一系列启动特定构建或交付工具的步骤，这些工具接收特定的输入并产生特定的输出。 |
| `TaskRun`     | 用特定的输入、输出和执行参数实例化一个`Task`以供执行。可以单独调用，也可以作为`Pipeline`的一部分调用。 |
| `Pipeline`    | 定义一系列完成特定构建或交付目标的`任务`。可以由事件触发或从`PipelineRun`调用。 |
| `PipelineRun` | 用特定的输入、输出和执行参数实例化一个`Pipeline`以供执行。   |

### 触发器(Triggers) 和 事件监听(EventListeners)

Tekton Triggers 包含在 Kubernetes 集群上运行的控制器服务以及 Kubernetes 自定义资源 (CRD)，它们扩展了 Tekton Pipelines 的功能以支持事件：

- [`EventListener`](https://tekton.dev/docs/triggers/eventlisteners/) - 在`Kubernetes`集群的指定端口监听事件。指定一个或多个`Triggers`。
- [`Trigger`](https://tekton.dev/docs/triggers/triggers/) - ` EventListener `检测到事件时会发生什么。一个`Trigger `指定一个`TriggerTemplate `，一个` TriggerBinding `，还有一个可选的` Interceptor `。
- [`TriggerTemplate`](https://tekton.dev/docs/triggers/triggertemplates/) - 在 `EventListener` 检测到事件时实例化或执行的资源（例如 `TaskRun` 或 `PipelineRun`）指定蓝图。它暴露了可以在模板资源中的任何位置使用的参数。
- [`TriggerBinding`](https://tekton.dev/docs/triggers/triggerbindings/) - 从`EventListeners` 的 `Trigger` 定义中的提取扩展的变量数据，以及填充相应的`TriggerTemplate`中定义的参数。然后`TriggerTemplate`根据参数值填充相关的`TaskRun`或`PipelineRun`中的参数。
- [`ClusterTriggerBinding`](https://tekton.dev/docs/triggers/triggerbindings/) - `TriggerBinding`的集群范围版本，特别适合在集群中重用。
- [`Interceptor`](https://tekton.dev/docs/triggers/interceptors/) - `Interceptor`是在 TriggerBinding 之前运行的特定平台的`catch-all`事件处理器。 它允许您执行有效数据过滤、验证（使用秘密）、转换、定义和测试触发条件，以及实现其他有用的处理。 一旦事件数据通过`Interceptor`，它就会在您将有效数据传递给` TriggerBinding` 之前进入`Trigger`。 您还可以使用`Interceptor`来修改关联`Trigger`的行为。

### 拉取模板代码

Tasks/Pipelines 可以从`https://hub.tekton.dev/` 中寻找已有项目参考

clone编写好的示例和模板, 期中部分tasks 是从`https://hub.tekton.dev/` 中下载修改的。

```sh
git clone https://github.com/paradeum-team/tekton-catalog.git
```

目录结构

```
tekton-catalog
├── eventListeners
│   └── gitlab-EventListener.yaml
├── LICENSE
├── pipelines
│   ├── golang.yaml
│   └── nodejs.yaml
├── pvcs
│   └── tekton-cache-claim.yaml
├── rbacs
│   └── tekton-triggers-eventlistener-rbac.yaml
├── README.md
├── tasks
│   ├── delete-pipelineruns.yaml
│   ├── dockerfile-lint.yaml
│   ├── git-cli.yaml
│   ├── git-clone.yaml
│   ├── golang-build.yaml
│   ├── kaniko.yaml
│   ├── nodejs-build.yaml
│   └── yq-helm-chart-version.yaml
├── triggers
│   ├── golang-triggerTemplate.yaml
│   └── nodejs-triggerTemplate.yaml
└── triggersBindings
    └── gitlab-nft-triggerBinding.yaml
```

### 创建 Tasks

```sh
cat tekton-catalog/tasks/*.yaml | kubectl -n tekton-pipelines create -f -
```

### 创建 Pipelines

示例 Pipelines 中的Pipeline 流程图

![pld-tekton-pipeline流程图](./images/pld-tekton-pipeline流程图.png)

执行创建pipelines

```sh
cat tekton-catalog/pipelines/*.yaml | kubectl -n tekton-pipelines create -f -
```

### 创建 pvcs

酌情修改pvcs 中的yaml 内容，创建cache pvc, `size 10Gi` , `storageClassName: manual`

```
cat tekton-catalog/pvcs/*.yaml | kubectl -n tekton-pipelines create -f -
```

### 创建 Triggers

创建EventListernets 和 Triggers使用的sa, 并授权

```sh
cat tekton-catalog/rbacs/*.yaml | kubectl -n tekton-pipelines create -f -
```

酌情修改并创建 TriggerBindings

```sh
kubectl -n tekton-pipelines apply -f tekton-catalog/triggersBindings/gitlab-nft-triggerBinding.yaml
```

TriggerTemplates根据 Pipeline 创建PipelineRun, 创建 TriggerTemplates

```sh
cat tekton-catalog/triggers/*.yaml | kubectl -n tekton-pipelines create -f -
```

### 创建 EventListerners

#### 创建tekton 访问 gitlab 的认证 secret

`gitlab-secret.yaml`

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: gitlab-basic-auth-secret
type: Opaque
stringData:
  .gitconfig: |
    [credential "http://gitlab.example.com"]
      helper = store
    [credential "https://gitlab.example.com"]
      helper = store
  .git-credentials: |
    http://cicd:xxxxxxxx@gitlab.example.com
    https://cicd:xxxxxxxx@gitlab.example.com
```

```sh
kubectl -n tekton-pipelines apply -f gitlab-secret.yaml
```

#### 创建tekton 访问registry 仓库的认证secret

config.json

```json
{"auths":{"registry.paradeum.com":{"username":"cicd","password":"xxxxxxxx","auth":"xxxxxxxxxxx"}}}
```

```
kubectl create secret generic pld-cicd-secret \
  -n tekton-pipelines \
  --from-file=config.json=./config.json \
  --type=kubernetes.io/dockerconfig
```

#### 项目代码集成webhook

创建 gitlab webhook 使用的 secret `gitlab-webhook-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-webhook-secret
  namespace: tekton-pipelines
type: Opaque
stringData:
  secretToken: "xxxxxx"
```

```sh
kubectl apply -f gitlab-webhook-secret.yaml
```

gitlab 项目中

`设置`-`集成` 输入tekton webhook地址

```
https://tekton-trigger-gitlab.solarfs.io
```

```
tekton webhook token
```

`Add webhook`

#### 酌情修改并创建EventListerners

示例跟默认是只监听了 项目 `Tag  push event`，Code Git Repo 打tag 会通知 `EventListerner`, `EventListerner` 再通过 `Interceptors`、`TriggerBindings`、`TriggerTempates` 生成对应`PipelineRun` 

```sh
kubectl -n tekton-pipelines apply -f tekton-catalog/eventListeners/gitlab-EventListener.yaml
```

## 参考

https://github.com/paradeum-team/tekton-catalog/

https://www.qikqiak.com/post/create-ci-pipeline-with-tekton-1/

https://www.bianchengquan.com/article/511721.html

https://www.qikqiak.com/k8strain2/devops/tekton/overview/

https://github.com/tektoncd/dashboard/blob/main/docs/install.md

https://github.com/tektoncd/pipeline/blob/v0.31.0/docs/install.md

https://github.com/tektoncd/triggers/blob/v0.16.0/docs/install.md

https://github.com/tektoncd/operator

https://tekton.dev/docs/getting-started/

https://tekton.dev/docs/triggers/install/

https://github.com/IBM/ibm-garage-tekton-tasks

https://github.com/tektoncd/catalog/

https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-running-pod/

http://hbchen.com/post/devops/2021-08-09-tekton-golang/

https://cloud.tencent.com/developer/article/1645052

https://www.51cto.com/article/668514.html

https://seankhliao.com/blog/12020-04-28-tekton-pipelines/

https://spex.top/archives/Kubernetes-cronjon-auto-cleanup-tekton-pipelinerun.html