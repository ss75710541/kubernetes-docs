# 使用virtualbox自定义Alpine-vrit云镜像
    
下载 [alpine-virt-3.11.5-x86_64.iso](http://dl-cdn.alpinelinux.org/alpine/v3.10/releases/x86_64/alpine-virt-3.11.5-x86_64.iso)

## 创建vm

virtualbox 创建vm操作参考 [在 VirtualBox 中创建新的虚拟机](https://www.virtualbox.org/manual/UserManual.html#intro-starting)

注意：默认virtbox 的引导顺序为 软驱，光驱，硬盘，所有创建vm 后，必须 在设置--》系统--》启动顺序中 把硬盘引导设置为第一位，否则以后系统就算安装到磁盘，默认还是进入内存引导了iso 系统，相当于始终没有持久化


## 安装 alpine

通过virtualbox console 界面登入vm, alpine 默认root无密码

## 设置网卡

```
# setup-interfaces

# rc-update add networking

# service networking restart

```

## 设置dns server

```
# setup-dns
```

DNS domain name 不输入直接回车

DNS nameservers 输入 `114.114.114.114`

## 设置apk repos

vi /etc/apk/repositories

内容如下：

```
/media/cdrom/apks
http://mirror.lzu.edu.cn/alpine/v3.11/main
```

## 设置持久化存储

alpine 默认从内存引导启动，没有持久化，所以需要执行下面操作

```
setup-disk
```

输入块设备 `sda`

输入持久化的数据类型 `sys`

## 关机

```
poweroff
```

## 删除 iso 引导项

注意：vitrualbox , 调整引导顺序，或删除 iso 引导， 否则默认还是会进入iso 引导到内存系统

## 开机

在vitrualbox 中启动

## 设置ssh

因为dns 信息重启会失效，所以设置ssh 前重新设置 dns

```
# setup-dns
```

```
# setup-sshd
```

开放root 远程ssh登录

vi /etc/ssh/sshd_config

找到PermitRootLogin，放开注释，修改内容如下

```
PermitRootLogin yes
```

重启sshd

```
service sshd restart
```

## 修改root 默认密码

```
passwd
```

## 替换时区文件 /etc/localtime

```
scp Shanghai root@192.168.99.107:/etc/localtime
```

## 安装 cloud-init

添加社区测试源

vi /etc/apk/repositories

```
#/media/cdrom/apks
http://mirror.lzu.edu.cn/alpine/v3.11/main
http://mirror.lzu.edu.cn/alpine/v3.11/community
@edge http://mirror.lzu.edu.cn/alpine/edge/main
@edgecommunity http://mirror.lzu.edu.cn/alpine/edge/community
@testing http://mirror.lzu.edu.cn/alpine/edge/testing
```


```
# apk add --update cloud-init@testing
# rm -rf /var/lib/apt/lists/*
# rm /var/cache/apk/*
# rc-update add cloud-init
# service cloud-init restart
```

## 关机

```
poweroff
```

## 导出vm

参考：[Exporting an Appliance in OVF Format
](https://www.virtualbox.org/manual/UserManual.html#ovf-export-appliance) 导出ovf 2.0 格式 文件

结果文件示例：alpine310-virt-base.ova

## 转换格式

安装 qemu-img, 使用qemu-img 转换格式

centos7 中安装 qeum-img 使用

```
yum install -y qeum-img
```

执行转换命令

```
VERSION=0.1
tar xvf alpine310-virt-base.ova
qemu-img convert -O qcow2 alpine310-virt-base-disk001.vmdk alpine310-virt-base-${VERSION}.qcow2
gzip alpine310-virt-base-${VERSION}.qcow2
```

最终文件为 `alpine310-virt-base-0.1.qcow2.gz`

## 发布到kubevirt

启动一个http服务，alpine310-virt-base-0.1.qcow2.gz 提供下载，示例下载地址为：

```
http://172.26.163.182:8081/packages/kubevirt-image/alpine310-virt-base-0.8.qcow2.gz
```

在 kube web ui 中以下面 yaml 创建vm

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: alpine310-virt
spec:
  dataVolumeTemplates:
    - metadata:
        name: alpine310-virt-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: custom-nas-storage
        source:
          http: 
            url: http://172.26.163.182:8081/packages/kubevirt-image/alpine310-virt-base-0.8.qcow2.gz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        vm.kubevirt.io/name: alpine310-virt
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: vm
                operator: In
                values:
                - 'true'
      domain:
        cpu:
          cores: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
          interfaces:
            - bootOrder: 2
              masquerade: {}
              name: nic0
          rng: {}
        machine:
          type: ''
        resources:
          requests:
            memory: 64M
      hostname: alpine310-virt
      networks:
        - name: nic0
          pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
        - dataVolume:
            name: alpine310-virt-rootdisk
          name: rootdisk
```
