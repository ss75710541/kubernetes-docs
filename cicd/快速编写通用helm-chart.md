# 快速编写通用helm chart

## helm 版本 v3.7+

## 创建helm chart

```shell
helm create <NAME>
```

## 修改 Chart.yaml

```yaml
version: 0.1.0 # chart版本
appVersion: "x.x.x" # chart 只有一个服务一般同镜像tag

# 如果需要发布在 https://artifacthub.io/, 酌情添加修改下面内容，不需要的可以忽略
home: https://github.com/xxxxxxxxxxxxx/xxxxxx
keywords:
  - xxxxxx
  - xxxxxx-chart
source:
  - https://github.com/xxxxxxxxxxxxx/xxxxxx
  - https://github.com/xxxxxxxxxxxxx/xxxxxx-chart
maintainers:
  - name: ss75710541
    email: 75710541@qq.com
    url: https://github.com/ss75710541
annotations:
  artifacthub.io/links: |
    - name: Chart Source
      url: https://github.com/xxxxxxxxxxxxx/xxxxxx-chart
    - name: Source
      url: https://github.com/xxxxxxxxxxxxx/xxxxxx
```

## 修改 values.yaml

```yaml
image:
  repository: nginx # 修改镜像地址
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "" # 修改tag

env: [] # 增加服务需要的 env 列表(非需要隐藏的env)，注意所有value 值全添加上引号 ""(字符或数字不加引号部署时有可能会异常)
  # - name: SERVER_PORT
  #   value: "8187"

envFrom: [] # 增加服务需要的从secret 读取的env 列表(需要隐藏的env,数据库用户密码等)
  #- secretRef:
  #  name: xxxx-secret

extraVolumeMounts: #[] # 增加挂载扩展挂载目录
  - mountPath: /etc/localtime
    name: localtime
    readOnly: true

extraVolumes: #[] # 增加扩展卷
  - name: localtime
    hostPath:
      path: /etc/localtime
```

### secret env 示例

dev-mysql.env

```
DATABASE_USER=username
DATABASE_PWD=password
DATABASE_DBNAME=dbname
```

```
kubectl create secret generic dev-mysql-secret --from-env-file=dev-mysql.env
```

## 修改 deployment.yaml

修改 templates/deployment.yaml

增加 env / envFrom / extraVolumeMounts / extraVolume 相关内容

```yaml
    ...
    spec:
      ...
      containers:
        - name: {{ .Chart.Name }}
          # 增加下面内容到 templates/deployment.yaml
          {{- with .Values.env }}
          env:
          {{- toYaml . | nindent 10 }}
          {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
          {{- toYaml . | nindent 10 }}
          {{- end }}
          ...
          # 酌情修改 port 值
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          # 酌情修改 path 值
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          {{- if .Values.extraVolumeMounts }}
          volumeMounts: {{- toYaml .Values.extraVolumeMounts | nindent 12 }}
          {{- end }}
      ...
      {{- if .Values.extraVolumes }}
      volumes: {{ toYaml .Values.extraVolumes | nindent 8 }}
      {{- end }}
```

## 使用helm-docs 自动生成helm chart REDAME.md

#### Mac 安装helm-docs

```sh
brew install norwoodj/tap/helm-docs
```

其它平台安装参考：https://github.com/norwoodj/helm-docs

#### 创建模板文件 `README.md.gotmpl`

````
{{ template "chart.header" . }}
{{ template "chart.description" . }}

{{ template "chart.versionBadge" . }}{{ template "chart.typeBadge" . }}{{ template "chart.appVersionBadge" . }}

## Installing the Chart

Add repository

```
helm repo add paradeum-team https://xxxxxxxxxxxxx.github.io/helm-charts/
helm repo update
```

Install chart

```
helm install my-walletconnect-relay xxxxxxxxxxx/xxxxxxxxxx --version --version {{ template "chart.version" . }}
```

{{ template "chart.requirementsSection" . }}

{{ template "chart.valuesSection" . }}
````

#### 生成 README.md

```sh
helm-docs .
```

