# k8s 使用coredns 自定义hosts 解析

##  修改coredns cm

```sh
kubectl edit cm coredns -n kube-system
```

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods verified
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . 10.8.255.1 10.8.255.2 8.8.8.8 {
           max_concurrent 1000
        }
        autopath @kubernetes
        cache 30
        loop
        reload
        loadbalance

				# 自定义解析内网解析
        hosts {
          172.16.187.71 liujinye.example.com
          fallthrough
        }
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

coredns 服务会自动加载配置

## 如果部署了node-local-dns， 同时也要修改 node-local-dns cm

```sh
kubectl edit cm -n kube-system node-local-dns
```

```yaml
apiVersion: v1
data:
  Corefile: |
    cluster.local:53 {
        errors
        cache {
                success 9984 30
                denial 9984 5
        }
        reload
        loop
        bind 169.254.20.10 __PILLAR__DNS__SERVER__
        forward . 10.96.0.10 {
                force_tcp
        }
        prometheus :9253
        health 169.254.20.10:8080
        }
    in-addr.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.20.10 __PILLAR__DNS__SERVER__
        forward . 10.96.0.10 {
                force_tcp
        }
        prometheus :9253
        }
    ip6.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.20.10 __PILLAR__DNS__SERVER__
        forward . 10.96.0.10 {
                force_tcp
        }
        prometheus :9253
        }
    .:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.20.10 __PILLAR__DNS__SERVER__
        #forward . __PILLAR__UPSTREAM__SERVERS__ # 线上dns解析默认是不通过coredns
        forward . 10.96.0.10 { # 修改为转发coredns解析
                force_tcp
        }
        prometheus :9253
        }
kind: ConfigMap
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
  name: node-local-dns
  namespace: kube-system
```

node-local-dns服务会自动加载配置