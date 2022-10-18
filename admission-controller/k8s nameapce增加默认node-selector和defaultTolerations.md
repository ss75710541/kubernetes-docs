# k8s nameapce增加默认node-selector和defaultTolerations

## 开启准入控制器

编辑所有master `/etc/kubernetes/manifests/kube-apiserver.yaml`,修改后kube-apiserver pod  会自动重启

```
spec:
  containers:
  - command:
    ...
    - --enable-admission-plugins=NodeRestriction,PodNodeSelector,PodTolerationRestriction
    ...
```

## 在namespace 注解中配置节点选择相关内容

```
apiVersion: v1
kind: Namespace
metadata:
  ...
	annotations:
    ...
    # 默认的节点选择
    scheduler.alpha.kubernetes.io/node-selector: 'node-role.kubernetes.io/whitetiger=true'
    # 默认的污点容忍
    scheduler.alpha.kubernetes.io/defaultTolerations: '[{"operator": "Exists", "effect": "NoSchedule", "key": "whitetiger"}]'
    # 污点容忍列表白名单
    scheduler.alpha.kubernetes.io/tolerationsWhitelist: '[{"effect":"NoExecute","key":"node.kubernetes.io/not-ready","operator":"Exists","tolerationSeconds":300},{"effect":"NoExecute","key":"node.kubernetes.io/unreachable","operator":"Exists","tolerationSeconds":300},{"effect":"NoSchedule","key":"whitetiger","operator":"Exists"}]'
  ...
```

namespace 配置好上面内容后，在该namespace 中发布的服务默认会添加对应的 节点选择 和 污点容忍

## 参考：

https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/