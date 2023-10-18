# argocd2.2.1+helm3.9-chart+k8s1.22部署flowise

## 创建使用的secret

flowise-secrets.env

```sh
flowisepassword=11111 # flowise web 用户密码
username=devflowise # psql 普通用户
password=11111 # psql 普通用户密码
passphrase=11111 # flowise 密码短语
postgres-password=11111 # psql postgres 用户密码
replication-password=11111 # psql 同步用户密码
```

```sh
kubectl create secret generic flowise-secrets --from-env-file=./flowise-secrets.env -n dev-flowise
```

## 部署postgresql

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-flowise-postgresql
  namespace: cicd
spec:
  destination:
    name: ucloud-hk-kugga-prod
    namespace: dev-flowise
  project: dev-flowise
  source:
    chart: postgresql
    helm:
      parameters:
      - name: architecture
        value: replication
      - name: auth.username
        value: devflowise
      - name: auth.database
        value: flowise
      - name: auth.existingSecret
        value: flowise-secrets
      - name: backup.enabled
        value: "true"
      - name: backup.cronjob.storage.storageClass
      - name: global.storageClass
        value: local-path
      # 因argocd v2.2.1 解析postgresql 不到helm chart 中 backup.cronjob.storage.storageClass 配置项所以弃用的这个配置
      #- name: backup.cronjob.storage.storageClass
      #  value: nfs4-client
      # 提前手动使用storageClass nfs4-client 创建了pvc dev-flowise-postgresql-primary-pgdumpall , 在这引用
      - name: backup.cronjob.storage.existingClaim
        value: dev-flowise-postgresql-primary-pgdumpall
      valueFiles:
      - values.yaml
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: 13.1.5
```

## 部署flowise

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-flowise
  namespace: cicd
spec:
  destination:
    name: ucloud-hk-kugga-prod
    namespace: dev-flowise
  project: dev-flowise
  source:
    chart: flowise
    helm:
      valueFiles:
      - values.yaml
      values: |-
        ## @param replicaCount Number of replicas (do not change it)
        replicaCount: 2

        ingress:
          enabled: true
          annotations:
            nginx.ingress.kubernetes.io/proxy-body-size: 100m
            nginx.ingress.kubernetes.io/upstream-hash-by: "$http_x_forwarded_for"
          ingressClassName: "whitetiger-nginx"
          pathType: ImplementationSpecific
          hosts:
            - host: gpt.solarfs.ai
              paths:
                - /
          tls:
            - secretName: solarfs-ai-tls
              hosts:
                - gpt.solarfs.ai

        persistence:
          enabled: true
          storageClass: nfs4-client

        ## @section Config parameters

        config:
          ## @param config.username Username to login
          username: "admin"

          ## @param config.password Password to login
          password: "admin"

          ## @param config.passphrase Passphrase used to create encryption key
          passphrase: MYPASSPHRASE

        ## @param existingSecret Name of existing Secret to use
        existingSecret: "flowise-secrets"

        ## @param existingSecretKeyPassword Key in existing Secret that contains password
        existingSecretKeyPassword: flowisepassword

        ## @param existingSecretKeyPassphrase Key in existing Secret that contains passphrase
        existingSecretKeyPassphrase: passphrase


        ## @section PostgreSQL parameters

        externalPostgresql:
          ## @param externalPostgresql.enabled Whether to use an external PostgreSQL
          enabled: true

          ## @param externalPostgresql.host External PostgreSQL host
          host: dev-flowise-postgresql-primary

          ## @param externalPostgresql.port External PostgreSQL port
          port: 5432

          ## @param externalPostgresql.username External PostgreSQL user
          username: devflowise

          ## @param externalPostgresql.password External PostgreSQL password
          password: flowise

          ## @param externalPostgresql.existingSecret Name of existing Secret to use
          existingSecret: "flowise-secrets"

          ## @param externalPostgresql.existingSecretKeyPassword Key in existing Secret that contains PostgreSQL password
          existingSecretKeyPassword: password

          ## @param externalPostgresql.database External PostgreSQL database
          database: flowise
    repoURL: https://cowboysysop.github.io/charts/
    targetRevision: 2.5.0
  syncPolicy:
    automated: {}
```

```sh
kubectl apply -f dev-flowise.yaml
```

