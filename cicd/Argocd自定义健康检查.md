# Argocd自定义健康检查

## 概述

argocd 健康检查 ingress 的标准是 status.loadBalancer.ingress 列表非空，至少有一个主机名或 IP 值。

ingress-nginx-controller 如果 使用 hostNetwork 方式部署, 添加 ingress 时会没有 ip 和 hostname，这处情况下就需要 自定义argocd 的 ingress 健康检查来解决

```
NAME                                 CLASS        HOSTS                 ADDRESS   PORTS     AGE
prod-blockscout                      geth-nginx   x.x.x             		80, 443   8h
prod-browser-node-geth-wss-ingress   geth-nginx   x.x.x                 80, 443   14d
```

## ingress 中添加注解

```
  ...
  annotations:
    argocd-ignore-health-check: "true"
  ...
```

## 修改 argocd-cm 增加自定义健康检查

```
kubectl -n argo-cd edit cm argocd-cm
```

Argo CD 使用用 Lua 编写的自定义健康检查。

自定义健康检查可以在 argocd-cm 的 resource.customizations.health.<group_kind> 字段中定义。以下示例是 networking.k8s.io_Ingress 的健康检查。

```yaml
data:
  ...
  resource.customizations.health.networking.k8s.io_Ingress: |
    hs = {}
    if obj.metadata.annotations ~= nil then
      for k,v in pairs(obj.metadata.annotations) do
        if k == "argocd-ignore-health-check" then
          hs.status = "Healthy"
          hs.message = "Ingress ignore health check"
          return hs
        end
      end
    end
    if obj.status ~= nil then
      if obj.status.loadBalancer ~= nil then
        if obj.status.loadBalancer.ingress ~= nil then
          hs.status = "Healthy"
          hs.message = "Ingress healthy"
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for ingerss"
    return hs
  resource.customizations.useOpenLibs.networking.k8s.io_Ingress: "true"
```

## 参考

https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#custom-health-checks