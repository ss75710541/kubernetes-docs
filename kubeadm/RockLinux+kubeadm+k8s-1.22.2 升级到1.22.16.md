# RockLinux+kubeadm+k8s-1.22.2 升级到1.22.16

## 升级第一个控制面

安装kubeadm 1.22.16

```
yum install -y kubeadm-1.22.16-0 --disableexcludes=kubernetes
```

验证 kubeadm 版本

```
kubeadm version
```

检查kubeadm-config 酌情修改

```sh
kubectl edit -n kube-system cm kubeadm-config
```

内容如下

```yaml
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
        enable-admission-plugins: NodeRestriction,PodNodeSelector,PodTolerationRestriction
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: api-server.solarfs.k8s:6443
    controllerManager:
      extraArgs:
        bind-address: 0.0.0.0
    dns:
      imageRepository: registry.hisun.netwarps.com/coredns
      imageTag: 1.8.0
    etcd:
      local:
        dataDir: /var/lib/etcd
        extraArgs:
          listen-client-urls: https://0.0.0.0:2379
          listen-metrics-urls: http://0.0.0.0:2381
          listen-peer-urls: https://0.0.0.0:2380
    imageRepository: registry.hisun.netwarps.com/google_containers
    kind: ClusterConfiguration
    kubernetesVersion: v1.22.2
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.128.0.0/16
      serviceSubnet: 10.96.0.0/12
    scheduler:
      extraArgs:
        bind-address: 0.0.0.0
```

验证升级计划：

```sh
kubeadm upgrade plan

# 显示如下
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.22.2
[upgrade/versions] kubeadm version: v1.22.16
I1123 19:20:20.171098   38597 version.go:255] remote version is much newer: v1.25.4; falling back to: stable-1.22
[upgrade/versions] Target version: v1.22.16
[upgrade/versions] Latest version in the v1.22 series: v1.22.16

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT        TARGET
kubelet     11 x v1.22.2   v1.22.16

Upgrade to the latest version in the v1.22 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.22.2   v1.22.16
kube-controller-manager   v1.22.2   v1.22.16
kube-scheduler            v1.22.2   v1.22.16
kube-proxy                v1.22.2   v1.22.16
CoreDNS                   1.8.0     v1.8.4
etcd                      3.5.0-0   3.5.0-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.22.16

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

执行升级到 v1.22.16

```
kubeadm upgrade apply v1.22.16
```

正常情况下显示如下

```
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.22.16". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

**手动升级你的 CNI 驱动插件**

你的容器网络接口（CNI）驱动应该提供了程序自身的升级说明。 参阅[插件](https://v1-23.docs.kubernetes.io/zh/docs/concepts/cluster-administration/addons/)页面查找你的 CNI 驱动， 并查看是否需要其他升级步骤。

如果 CNI 驱动作为 DaemonSet 运行，则在其他控制平面节点上不需要此步骤。

**flannel v0.15.1 升级到 v0.20.1**

下载flannel.yml

```sh
curl -o kube-flannel-v0.20.1.yaml  https://raw.githubusercontent.com/flannel-io/flannel/v0.20.1/Documentation/kube-flannel.yml
```

酌情修改配置

```
  ...
  net-conf.json: |
    {
      "Network": "10.128.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
   ...
```

创建flannel v0.20.1 

```sh
kubectl apply -f kube-flannel-v0.20.1.yaml
```

删除在 namespace kube-system 中旧 kube-flannel-ds daemonset

```sh
kubectl delete ds kube-flannel-ds -n kube-system
```

**对比服务配置**

升级前的配置备份在 `/etc/kubernetes/tmp/kubeadm-backup-manifests-2022-11-23-20-46-37`,有可能手动修改过的服务配置，酌情使用kubeadm重新修改相关配置，参考：[重新配置 kubeadm 集群](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/)

```sh
diff /etc/kubernetes/tmp/kubeadm-backup-manifests-2022-11-23-20-46-37/kube-controller-manager.yaml /etc/kubernetes/manifests/kube-controller-manager.yaml
diff /etc/kubernetes/tmp/kubeadm-backup-manifests-2022-11-23-20-46-37/kube-scheduler.yaml /etc/kubernetes/manifests/kube-scheduler.yaml
diff /etc/kubernetes/tmp/kubeadm-backup-manifests-2022-11-23-20-46-37/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
diff /etc/kubernetes/tmp/kubeadm-backup-manifests-2022-11-23-20-46-37/etcd.yaml /etc/kubernetes/manifests/etcd.yaml
```

**对于其它控制面节点**

与第一个控制面节点相同，但是使用：

```shell
kubeadm upgrade node
```

而不是：

```shell
kubeadm upgrade apply
```

此外，不需要执行 `kubeadm upgrade plan` 和更新 CNI 驱动插件的操作。

## 腾空节点

- 通过将节点标记为不可调度并腾空节点为节点作升级准备：

  ```shell
  # 将 <node-to-drain> 替换为你要腾空的控制面节点名称
  kubectl drain <node-to-drain> --ignore-daemonsets
  ```

### 升级 kubelet 和 kubectl

- 升级 kubelet 和 kubectl：

```sh
 yum install -y kubelet-1.22.16-0 kubectl-1.22.16-0 --disableexcludes=kubernetes
```

- 重启 kubelet

  ```shell
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  ```

### 解除节点的保护

- 通过将节点标记为可调度，让其重新上线：

  ```shell
  # 将 <node-to-drain> 替换为你的节点名称
  kubectl uncordon <node-to-drain>
  ```

## 升级工作节点

工作节点上的升级过程应该一次执行一个节点，或者一次执行几个节点， 以不影响运行工作负载所需的最小容量。

### 升级 kubeadm

```sh
yum install -y kubeadm-1.22.16-0 --disableexcludes=kubernetes
```

### 执行 "kubeadm upgrade"

- 对于工作节点，下面的命令会升级本地的 kubelet 配置：

  ```shell
  sudo kubeadm upgrade node
  ```

### 腾空节点

- 将节点标记为不可调度并驱逐所有负载，准备节点的维护：

  ```shell
  # 将 <node-to-drain> 替换为你正在腾空的节点的名称
  kubectl drain --ignore-daemonsets <node-to-drain> 
  ```

### 升级 kubelet 和 kubectl

- 升级 kubelet 和 kubectl：

  ```shell
  # 将 1.22.16-0 x 替换为最新的补丁版本
  yum install -y kubelet-1.22.16-0 kubectl-1.22.16-0 --disableexcludes=kubernetes
  ```

- 重启 kubelet

  ```shell
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  ```

### 取消对节点的保护

- 通过将节点标记为可调度，让节点重新上线:

  ```shell
  # 将 <node-to-drain> 替换为当前节点的名称
  kubectl uncordon <node-to-drain>
  ```

## 验证集群的状态

在所有节点上升级 kubelet 后，通过从 kubectl 可以访问集群的任何位置运行以下命令， 验证所有节点是否再次可用：

```shell
kubectl get nodes
```

`STATUS` 应显示所有节点为 `Ready` 状态，并且版本号已经被更新。

`STATUS` 有 `NotReady` 状态节点

删除对应节点 kube-flannel-ds pod, 触发重启kube-flannel-ds pod 后正常

```sh
kubectl delete pod kube-flannel-ds-h7wjj -n kube-flannel
```

## 参考：

https://v1-23.docs.kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/

https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/control-plane-flags/

https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-controller-manager/

