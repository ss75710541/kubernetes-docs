# k8s nameapce增加默认node-selector和defaultTolerations

## 开启准入控制器

编辑 kubadm-config

```sh
kubectl edit cm -n kube-system kubeadm-config
```

内容参考如下

```yaml
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
        # 增加配置，开启 enable-admission-plugins
        enable-admission-plugins: NodeRestriction,PodNodeSelector,PodTolerationRestriction
        ...
```

备份 `/etc/kubernetes/`（必须）

编辑 kubeadm-init.yaml, 增加 

```yaml
...
apiServer:
  timeoutForControlPlane: 4m0s
  # 增加扩展配置
  extraArgs:
    authorization-mode: Node,RBAC
    enable-admission-plugins: NodeRestriction,PodNodeSelector,PodTolerationRestriction
    ...
```

要在 `/etc/kubernetes/manifests` 中编写新的清单文件，你可以使用：

```sh
# kubeadm init phase control-plane <component-name> --config <config-file>
kubeadm init phase control-plane apiserver --config kubeadm-init.yaml
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
    scheduler.alpha.kubernetes.io/tolerationsWhitelist: '[{"effect":"NoExecute","key":"node.kubernetes.io/not-ready","operator":"Exists"},{"effect":"NoExecute","key":"node.kubernetes.io/unreachable","operator":"Exists"},{"effect":"NoSchedule","key":"node.kubernetes.io/memory-pressure","operator":"Exists"},{"effect":"NoSchedule","key":"whitetiger","operator":"Exists"}]'
  ...
```

namespace 配置好上面内容后，在该namespace 中发布的服务默认会添加对应的 节点选择 和 污点容忍

## 参考：

https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/

https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/