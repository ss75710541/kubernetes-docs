# 使用argocd-notifications通知Tekton触发api-test

## 安装argocd-notifications

### 下载helm chart

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm pull argo/argocd-notifications
```

### 创建values.yaml

```yaml
argocdUrl: https://argo-argocd-server.cicd.svc
image:
  # build image by https://github.com/paradeum-team/argocd-notifications/tree/jyliu
  # fix: repo.GetAppDetails().Helm.parameters not get argocd override value 
  repository: quay.io/netwarps/argocd-notifications
  tag: v1.2.1.1
secret:
  items:
    webhooks-tekton-api-test-token: "xxxxxxxxxx"
    email-username: cicd@example.com
    email-password: xxxxxxxxx
logLevel: info
extraArgs:
  - --argocd-repo-server=argo-argocd-repo-server:8081

notifiers:
  service.email: |
    host: smtp.exmail.qq.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password
  service.webhook.tekton-api-test: |
    url: http://el-github-listener.tekton-pipelines.svc:8080
    headers:
      - name: X-Github-Token
        value: $webhooks-tekton-api-test-token
      - name: content-type
        value: "application/json"
subscriptions:
  - recipients:
    - webhook.tekton-api-test:""
    - email:liujinye@example.com
    triggers:
    - app-sync-succeeded
templates:
  template.app-sync-succeeded: |
    email:
      subject: Application {{.app.metadata.name}} has been successfully synced.
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} has been successfully synced at {{.app.status.operationState.finishedAt}}.
      Sync operation details are available at: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true .
    webhook:
      tekton-api-test:
        method: POST
        path: /
        body: |
          {
            "project": "{{.app.spec.project}}",
            "name": "{{.app.metadata.name}}",
            "envs": "{{ (call .repo.GetAppDetails).Helm.GetParameterValueByName "test.envs" }}",
            "imageurl": "{{ (call .repo.GetAppDetails).Helm.GetParameterValueByName "test.image.repository" }}",
            "imagetag": "{{ (call .repo.GetAppDetails).Helm.GetParameterValueByName "test.image.tag"}}",
            "command": "{{ (call .repo.GetAppDetails).Helm.GetParameterValueByName "test.command[0]"}}"
          }
triggers:
  trigger.on-sync-succeeded: |
    - description: Application syncing has succeeded
      send:
      - app-sync-succeeded
      when: app.status.operationState.phase in ['Succeeded']
defaultTriggers: |
  - on-sync-succeeded
```

### 安装argocd-notifications

```sh
helm upgrade --install argocd-notifications argocd-notifications-1.8.0.tgz -n cicd -f values.yaml
```

## 需要集成api test 的项目增加api test 相关服务及配置

### 需要集成api test 的项目（示例wine） helm chart 增加 test 相关默认变量

```yaml
test:
  parallelism: 1
  activeDeadlineSeconds: 100
  # 必须，api test 的镜像信息
  image:
    repository: registry.example.com/nft/wine-api-test
    tag: "v0.1.5"
  # 必须，tekton env-job 根据command[0]执行操作
  command:
    - "./wine-api-test"
  # 必须，notify tekton api test envs, 使用argocd 通知 tekton 使用 envs， tekton enb-job task 解析envs 转换为环境变量
  envs: "HOSTURL=https://example.com|BFSSERVER_HOSTURL=https://dev-wine-api.example.com"
  # helm test env, 手动使用helm安装时用下面env, argocd 部署时无用
  env:
    - name: HOSTURL
      value: "https://example.com"
    - name: BFSSERVER_HOSTURL
      value: "http://dev-wine-api.nft.svc"
```

### argocd wine项目中 修改 PARAMETERS 变量 test.envs 值

```yaml
test.envs: "HOSTURL=https://example.com|BFSSERVER_HOSTURL=https://test-wine-api.example.com"
```

### argocd 项目中添加注解订阅通知,在sync 成功时触发webhook tekton-api-test

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    # argocd app 订阅发邮件到 liujinye@example.com
		notifications.argoproj.io/subscribe.on-sync-succeeded.email: "liujinye@example.com"
    # argocd app 订阅通知 webhook tekton-api-test
		notifications.argoproj.io/subscribe.on-sync-succeeded.tekton-api-test: ""
```

## wine-api-test 项目集成Tekton

1. wine-api-test 代码添加tekton webhook 集成

2. 推送`Tag push event` 触发Tekton pipeline 制作 `wine-api-test` 镜像

3. Tekton pipeline 自动修改 `wine-api` helm chart 中 `test.image.tag` 值

参考：https://liujinye.gitbook.io/kubernetes-docs/cicd/shi-yong-tekton-gou-jian-ci-liu-cheng

## Tekton 配置 envs job webhook pipline 接收argocd-notifications 通知

负责接收argocd-notifications 的通知及触发执行api test pipeline

略

参考：

https://liujinye.gitbook.io/kubernetes-docs/cicd/shi-yong-tekton-gou-jian-ci-liu-cheng

https://github.com/paradeum-team/tekton-catalog

## 参考：

https://argocd-notifications.readthedocs.io/en/stable/

https://argocd-notifications.readthedocs.io/en/stable/templates/

https://github.com/argoproj/argo-helm/tree/master/charts/argocd-notifications

https://liujinye.gitbook.io/kubernetes-docs/cicd/shi-yong-tekton-gou-jian-ci-liu-cheng