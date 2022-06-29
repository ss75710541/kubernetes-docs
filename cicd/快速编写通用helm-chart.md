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

增加 env / envForm / extraVolumeMounts / extraVolume 相关内容

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

