# ingress-nginx启用header名称中下划线

## 描述

默认ingress-nginx 会过滤掉header中带下划线 '_'  的header, header名称不建议使用下划线

## 方法一

修改代码，带下划线的修改为中横线'_'

## 方法二

configmap中开启 `enable-underscores-in-headers` 参数, 默认 false

```yaml
controller:
  ...
  config:
    ...
    enable-underscores-in-headers: "true"
    ...
```



## 参考：

https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#enable-underscores-in-headers

