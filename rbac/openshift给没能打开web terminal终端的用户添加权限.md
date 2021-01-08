# openshift给没能打开web terminal终端的用户添加权限 

创建 `role-pods-exec.yaml` 内容如下：

```
apiVersion: authorization.openshift.io/v1
kind: ClusterRole
metadata:
  name: pods-exec
rules:
- apiGroups:
  - ""
  attributeRestrictions: null
  resources:
  - pods/exec
  verbs:
  - get
```

创建cluster role pods-exec 

```
oc cretete -f role-exec.yaml
```

添加cluster role role-exec 权限 到 testexec 用户

```
oc adm policy add-cluster-role-to-user pods-exec testexec
```
