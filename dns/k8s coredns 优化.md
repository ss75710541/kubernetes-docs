# k8s coredns 优化

# 配置 NodeLocal DNS Cache

**说明：**

NodeLocal DNSCache 的本地侦听 IP 地址可以是任何地址，只要该地址不和你的集群里现有的 IP 地址发生冲突。 推荐使用本地范围内的地址，例如，IPv4 链路本地区段 '169.254.0.0/16' 内的地址， 或者 IPv6 唯一本地地址区段 'fd00::/8' 内的地址。

- 下载 nodelocaldns.yaml

```sh
wget https://raw.githubusercontent.com/kubernetes/kubernetes/v1.22.16/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

- 如果使用 IPv6，在使用 'IP:Port' 格式的时候需要把 CoreDNS 配置文件里的所有 IPv6 地址用方括号包起来。 如果你使用上述的示例清单， 需要把[配置行 L70](https://github.com/kubernetes/kubernetes/blob/b2ecd1b3a3192fbbe2b9e348e095326f51dc43dd/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml#L70) 修改为： "`health [__PILLAR__LOCAL__DNS__]:8080`"。

- 把清单里的变量更改为正确的值：

```sh
#kubedns=`kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}`
#domain=<cluster-domain>
#localdns=<node-local-address>
kubedns=`kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}`
domain=cluster.local
localdns=169.254.20.10
```

`<cluster-domain>` 的默认值是 "`cluster.local`"。`<node-local-address>` 是 NodeLocal DNSCache 选择的本地侦听 IP 地址。

- 修改 image repo

  ```sh
  sed -i 's/k8s.gcr.io/registry.solarfs.io/g' nodelocaldns.yaml
  ```

- 如果 kube-proxy 运行在 IPTABLES 模式：

  ```sh
  sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml
  ```

node-local-dns Pod 会设置 `__PILLAR__CLUSTER__DNS__` 和 `__PILLAR__UPSTREAM__SERVERS__`。 在此模式下, node-local-dns Pod 会同时侦听 kube-dns 服务的 IP 地址和 `<node-local-address>` 的地址，以便 Pod 可以使用其中任何一个 IP 地址来查询 DNS 记录。

- 如果 kube-proxy 运行在 IPVS 模式：

  ```sh
  sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,__PILLAR__DNS__SERVER__//g; s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml
  ```

在此模式下，node-local-dns Pod 只会侦听 `<node-local-address>` 的地址。 node-local-dns 接口不能绑定 kube-dns 的集群 IP 地址，因为 IPVS 负载均衡使用的接口已经占用了该地址。 node-local-dns Pod 会设置 `__PILLAR__UPSTREAM__SERVERS__`。

- 运行 `kubectl create -f nodelocaldns.yaml`

- 如果 kube-proxy 运行在 IPVS 模式，需要修改 kubelet 的 `--cluster-dns` 参数 NodeLocal DNSCache 正在侦听的 `<node-local-address>` 地址。 否则，不需要修改 `--cluster-dns` 参数，因为 NodeLocal DNSCache 会同时侦听 kube-dns 服务的 IP 地址和 `<node-local-address>` 的地址。

  Rock Linux 修改 `/etc/sysconfig/kubelet`

  ```sh
  KUBELET_EXTRA_ARGS="--cluster-dns=169.254.20.10"
  ```

  重启 `kubelet`

  ```sh
  systemctl restart kubelet
  ```

启用后，`node-local-dns` Pod 将在每个集群节点上的 `kube-system` 名字空间中运行。 此 Pod 在缓存模式下运行 [CoreDNS](https://github.com/coredns/coredns)， 因此每个节点都可以使用不同插件公开的所有 CoreDNS 指标。

如果要禁用该功能，你可以使用 `kubectl delete -f <manifest>` 来删除 DaemonSet。 你还应该回滚你对 kubelet 配置所做的所有改动。

## 优化 ndots

DNS 查询可能因为执行查询的 Pod 所在的名字空间而返回不同的结果。 不指定名字空间的 DNS 查询会被限制在 Pod 所在的名字空间内。 要访问其他名字空间中的 Service，需要在 DNS 查询中指定名字空间。

例如，假定名字空间 `test` 中存在一个 Pod，`prod` 名字空间中存在一个服务 `data`。

Pod 查询 `data` 时没有返回结果，因为使用的是 Pod 的名字空间 `test`。

Pod 查询 `data.prod` 时则会返回预期的结果，因为查询中指定了名字空间。

DNS 查询可以使用 Pod 中的 `/etc/resolv.conf` 展开。kubelet 会为每个 Pod 生成此文件。例如，对 `data` 的查询可能被展开为 `data.test.svc.cluster.local`。 `search` 选项的取值会被用来展开查询。要进一步了解 DNS 查询，可参阅 [`resolv.conf` 手册页面](https://www.man7.org/linux/man-pages/man5/resolv.conf.5.html)。

```
nameserver 169.254.20.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

概括起来，名字空间 `test` 中的 Pod 可以成功地解析 `data.prod` 或者 `data.prod.svc.cluster.local`。

### pod 中修改 ndots 

启动的服务pod 中增加 dnsConfig 配置, 减少search 查询数量

```yaml
  ...
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```

## 启用autopath 插件

autopath是一个可选的优化插件，可以提高集群外部名称查询的性能。启用autopath插件需要CoreDNS使用更多的内存来存储有关Pod的信息。Autopath的原理是在第一次域名查询失败尝试找到正确的域名，这样共需要2次（1次IPv4，1次IPv6）查询就可以获取到正确的结果, 减少客户端在查找外部名称时进行的DNS查询次数。

编辑 coredns configmap

```sh
kubectl edit configmap coredns -n kube-system
```

内容如下：

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
           pods verified # 修改这里
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        autopath @kubernetes # 增加这里
        prometheus :9153
        forward . 10.42.255.1 10.42.255.2 114.114.114.114 {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## 性能测试

安装 dnsperf

```sh
yum install dnsperf
```

准备好压测的dns 列表， dns.txt

```
www.baidu.com A
```

执行压测

```sh
dnsperf -l 30 -s 10.96.0.10 -d dns.txt
```

```
DNS Performance Testing Tool
Version 2.7.1

[Status] Command line: dnsperf -l 30 -s 10.96.0.10 -d dns.txt
[Status] Sending queries (to 10.96.0.10:53)
[Status] Started at: Sun Dec  4 13:09:04 2022
[Status] Stopping after 30.000000 seconds
[Status] Testing complete (time limit)

Statistics:

  Queries sent:         1703257
  Queries completed:    1703257 (100.00%)
  Queries lost:         0 (0.00%)

  Response codes:       NOERROR 1703257 (100.00%)
  Average packet size:  request 47, response 92
  Run time (s):         30.000809
  Queries per second:   56773.702336

  Average Latency (s):  0.001483 (min 0.000019, max 0.025491)
  Latency StdDev (s):   0.001063

```

```sh
dnsperf -l 30 -s 169.254.20.10 -d dns.txt
```

```
DNS Performance Testing Tool
Version 2.7.1

[Status] Command line: dnsperf -l 30 -s 169.254.20.10 -d dns.txt
[Status] Sending queries (to 169.254.20.10:53)
[Status] Started at: Sun Dec  4 13:10:16 2022
[Status] Stopping after 30.000000 seconds
[Status] Testing complete (time limit)

Statistics:

  Queries sent:         1623316
  Queries completed:    1623316 (100.00%)
  Queries lost:         0 (0.00%)

  Response codes:       NOERROR 1623316 (100.00%)
  Average packet size:  request 47, response 92
  Run time (s):         30.000978
  Queries per second:   54108.769387

  Average Latency (s):  0.001626 (min 0.000023, max 0.028625)
  Latency StdDev (s):   0.000950
```



## 参考

https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/nodelocaldns/

https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/

https://aws.amazon.com/cn/blogs/china/how-to-optimize-amazon-eks-cluster-dns-performance/

https://imroc.cc/tke/faq/install-localdns-with-ipvs/

https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

https://cloud.tencent.com/developer/article/1583706

https://imroc.cc/k8s/best-practice/optimize-dns/