# coredns service 连接超时(connection timed out; no servers could be reached)

## node 主机测试kube-dns 解析提示连接超时

```sh
dig @10.96.0.10 kube-dns.kube-system.svc.cluster.local

# 显示如下
; <<>> DiG 9.11.26-RedHat-9.11.26-6.el8 <<>> @10.96.0.10 kube-dns.kube-system.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

## 修改 coredns configmap 配置，查看coredns 日志

```sh
kubectl -n kube-system edit cm coredns
```

在 Corefile 增加`log` 配置

```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
...
```

保存更改后，k8s 将这些更改传播到CoreDNS豆荚可能需要一到两分钟的时间。CoreDNS会自动加载配置。

查询coredns 所有pod日志

```sh
kubectl logs coredns-5f9bcf7c57-hbq5z -n kube-system --tail 100
```

显示类似如下内容，没有异常信息

```
[INFO] 10.128.8.110:40481 - 26090 "A IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local.svc.cluster.local. udp 91 false 512" NXDOMAIN qr,aa,rd 184 0.000058542s
[INFO] 10.128.8.110:33032 - 61678 "A IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local. udp 73 false 512" NOERROR qr,aa,rd 144 0.000067122s
[INFO] 10.128.3.156:54546 - 33121 "A IN dev-mgo-nft-mongodb.nft.svc.nft.svc.cluster.local. udp 67 false 512" NXDOMAIN qr,aa,rd 160 0.000404809s
[INFO] 10.128.3.156:35509 - 17309 "A IN dev-mgo-nft-mongodb.nft.svc.svc.cluster.local. udp 63 false 512" NXDOMAIN qr,aa,rd 156 0.000294607s
[INFO] 10.128.3.156:41908 - 8575 "AAAA IN dev-mgo-nft-mongodb.nft.svc.cluster.local. udp 59 false 512" NOERROR qr,aa,rd 152 0.000117004s
[INFO] 10.128.8.110:59090 - 31404 "AAAA IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local.svc.cluster.local. udp 91 false 512" NXDOMAIN qr,aa,rd 184 0.000143273s
[INFO] 10.128.8.110:45311 - 21795 "AAAA IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local. udp 73 false 512" NOERROR qr,aa,rd 166 0.000079302s
[INFO] 10.128.8.110:42297 - 55594 "A IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local.svc.cluster.local. udp 91 false 512" NXDOMAIN qr,aa,rd 184 0.000044621s
[INFO] 10.128.8.110:58692 - 11604 "A IN kafka-zk-zookeeper-headless.zookeeper.svc.cluster.local. udp 73 false 512" NOERROR qr,aa,rd 144 0.000041031s
```

## 查看 svc 和 endpoints信息

```
# kubectl -n kube-system get endpoints kube-dns
NAME       ENDPOINTS                                                   AGE
kube-dns   10.128.0.58:53,10.128.6.152:53,10.128.0.58:53 + 3 more...   15h
[root@master1 ~]# kubectl -n kube-system describe endpoints kube-dns
Name:         kube-dns
Namespace:    kube-system
Labels:       k8s-app=kube-dns
              kubernetes.io/cluster-service=true
              kubernetes.io/name=CoreDNS
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2022-11-24T10:57:23Z
Subsets:
  Addresses:          10.128.0.58,10.128.6.152
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    dns-tcp  53    TCP
    dns      53    UDP
    metrics  9153  TCP

Events:  <none>
```



使用dig 测试 pod ip ,查看解析

```sh
 dig @10.128.0.58 kube-dns.kube-system.svc.cluster.local
 dig @10.128.6.152 kube-dns.kube-system.svc.cluster.local
```

结果都是正常的, 说明coredns 服务本身没有问题

```
; <<>> DiG 9.11.26-RedHat-9.11.26-6.el8 <<>> @10.128.0.58 kube-dns.kube-system.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41299
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 75dd2e6ddd4a4454 (echoed)
;; QUESTION SECTION:
;kube-dns.kube-system.svc.cluster.local.	IN A

;; ANSWER SECTION:
kube-dns.kube-system.svc.cluster.local.	30 IN A	10.96.0.10

;; Query time: 0 msec
;; SERVER: 10.128.0.58#53(10.128.0.58)
;; WHEN: Fri Nov 25 10:18:28 CST 2022
;; MSG SIZE  rcvd: 133
```

## telnet 测试连通性

使用telnet 在master 主机和 node 测试 10.96.0.10 53 端口，结果都是正常的

```sh
telnet 10.96.0.10 53
# 显示如下
Trying 10.96.0.10...
Connected to 10.96.0.10.
Escape character is '^]'.
Connection closed by foreign host.
```

## 根据网上搜索相关错误联系刚升级的flannel服务配置解决问题

根据 https://github.com/coredns/coredns/issues/3704 中信息提示，pod 网络和服务网络重叠会导致路由问题。

我猜测 可能跟我 刚更新的 cni  flannel 版本到v0.20.1 有关系

 flannel Network 配置如下

```yaml
  net-conf.json: |
    {
      "Network": "10.128.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

Kubeadm-config 中Networking配置如下

```yaml
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet:  10.128.0.0/16
```

Kubeadm-config 中 serviceSubnet 不在  flannel Network 配置范围中，我修改了 flannel Network 配置如下

```yaml
  net-conf.json: |
    {
      "Network": "10.0.0.0/8",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

重启flannel ds

```sh
kubectl rollout restart ds kube-flannel-ds -n kube-flannel
```

重新 使用dig 测试 kube-dns 域名

```sh
dig @10.96.0.10 kube-dns.kube-system.svc.cluster.local
```

可以正常解析

```
; <<>> DiG 9.11.26-RedHat-9.11.26-6.el8 <<>> @10.96.0.10 kube-dns.kube-system.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20581
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a7918abdce60333a (echoed)
;; QUESTION SECTION:
;kube-dns.kube-system.svc.cluster.local.	IN A

;; ANSWER SECTION:
kube-dns.kube-system.svc.cluster.local.	30 IN A	10.96.0.10

;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri Nov 25 10:33:40 CST 2022
;; MSG SIZE  rcvd: 133
```

## 参考：

https://medium.com/geekculture/k8s-troubleshooting-how-to-debug-coredns-issues-724e8b973cfc

https://github.com/coredns/coredns/issues/3704

https://coredns.io/manual/configuration/

https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/dns-custom-nameservers/

https://github.com/kubernetes-sigs/kubespray/issues/4674

https://github.com/coredns/deployment/blob/master/kubernetes/Upgrading_CoreDNS.md

https://github.com/flannel-io/flannel/blob/v0.20.1/Documentation/configuration.md