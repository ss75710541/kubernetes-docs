# Argocd定时备份到us3

## 创建secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-backup-secret
  namespace: cicd
stringData:
  accessKeyID: TOKEN_83cf9130-2884-4c6f-9d07-xxxxxxxxxxxxx # 为US3的令牌公钥。
  secretAccessKey: 7fe4a519-23a6-4c42-9665-xxxxxxxxxx # 为US3的令牌私钥。
  endpoint: http://internal.s3-hk.ufileos.com
  bucket: argocd-backup
```

```sh
kubectl apply -f argocd-backup-secret.yaml
```

## 创建定时任务

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  namespace: cicd
  name: argocd-backup
spec:
  schedule: "0 19 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            backup: "argocd"
        spec:
          initContainers:
          - name: argocd-backup
            image: quay.io/argoproj/argocd:v2.2.1
            env:
              - name: NODE_IP
                valueFrom:
                  fieldRef:
                    fieldPath: status.hostIP
              - name: ADMIN_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: argocd-initial-admin-secret
                    key: password
            args:
              - |
                #!/bin/sh
                set -e
                argocd --insecure login argo-argocd-server.cicd.svc --username admin --password $ADMIN_PASSWORD
                argocd admin export -n cicd|gzip > /backup/argocd-export-`date +%Y%m%d%H%M%S`.yaml.gz
                ls -l /backup/*.yaml.gz
            command:
              - /bin/sh
              - '-c'
            volumeMounts:
              - name: backup
                mountPath: /backup
          containers:
          - name: argocd-upload
            image: minio/mc:RELEASE.2022-12-13T00-23-28Z
            env:
              - name: ACCESS_KEY_ID
                valueFrom:
                  secretKeyRef:
                    name: argocd-backup-secret
                    key: accessKeyID
              - name: SECRET_ACCESS_KEY
                valueFrom:
                  secretKeyRef:
                    name: argocd-backup-secret
                    key: secretAccessKey
              - name: ENDPOINT
                valueFrom:
                  secretKeyRef:
                    name: argocd-backup-secret
                    key: endpoint
              - name: BUCKET
                valueFrom:
                  secretKeyRef:
                    name: argocd-backup-secret
                    key: bucket
            args:
              - |
                #!/bin/sh
                set -e
                mc config host add s3 $ENDPOINT $ACCESS_KEY_ID $SECRET_ACCESS_KEY --api s3v4
                # us3 目前只兼容 8M 分片上传，mc 客户端为16M分片，所以禁用了分片上传
                mc cp --disable-multipart /backup/argocd-export-* s3/$BUCKET/
            command:
              - /bin/sh
              - '-c'
            volumeMounts:
              - name: backup
                mountPath: /backup
          restartPolicy: OnFailure
          serviceAccount: argocd-application-controller
          volumes:
            - name: backup
              emptyDir: {}
```

创建CronJob

```sh
kubectl apply -f backup-argocd.yaml
```

