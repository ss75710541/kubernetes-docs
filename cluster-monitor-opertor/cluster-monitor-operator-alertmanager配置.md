# cluster-monitor-operator alertmanager配置

## 导出现有配置

```
kubectl -n openshift-monitoring get secret alertmanager-main -ojson | jq -r '.data["alertmanager.yaml"]' | base64 -d > alertmanager.yaml
```

编辑alertmanager.yaml

```
global:
  resolve_timeout: 5m
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: default
  routes:
  - match:
      alertname: DeadMansSwitch
    repeat_interval: 5m
    receiver: deadmansswitch
receivers:
- name: default
  email_configs:
  - to: 'liujinye@hisuntech.com'
    from: 'cicd@email.prardeum.com'
    smarthost: 'smtpdm.aliyun.com:465'
    auth_username: 'cicd@email.prardeum.com'
    auth_password: 'xxxxx'
    auth_secret: 'cicd@email.prardeum.com'
    auth_identity: 'cicd@email.prardeum.com'
- name: deadmansswitch
```

## 编写完成后，应用配置

```
kubectl -n openshift-monitoring create secret generic alertmanager-main --from-literal=alertmanager.yaml="$(< alertmanager.yaml)" --dry-run -oyaml | kubectl -n openshift-monitoring replace secret --filename=-
```
