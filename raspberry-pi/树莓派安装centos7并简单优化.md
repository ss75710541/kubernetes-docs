# 树莓派安装centos7并简单优化

## 硬件准备

硬件|数量
-----|-----
树莓派4代 |1
USB SSD硬盘 500G|1
TF卡 128G|1
TF卡读卡器|1
HDMI 转 micro HDMI 线|1
USB 键盘|1
支持HDMI接口的显示器|1
mac 或 windows 电脑|1

## 安装系统

### 下载系统镜像

在mac 或 windows 电脑 从下面地址下载镜像CentOS7 armv7 32位镜像到本地

[CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-4-1908-sda.raw.xz](http://mirrors.huaweicloud.com/centos-altarch/7.7.1908/isos/armhfp/CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-4-1908-sda.raw.xz)

### 烧录镜像到TF卡

TF卡插入 TF读卡器，TF读卡器插入 mac 或 windows电脑

打开网站 https://www.balena.io/etcher/，从中下载烧录镜像软件，会自动判断当前电脑系统，提供对应版本的软件Download链接

下载安装好balenaEtcher, 后运行

第一步选择镜像文件，第二步选择USB存储(USB读卡器+TF卡)，第三步点Flash

### 树莓派开机

- ssd 硬盘通过usb 3.0 接口(蓝色)连接树莓派
- 烧录好的TF卡插入树莓派
- USB 键盘插入树莓派
- HDMI 转 micro HDMI 线 ，分别连接  操作电脑 和 树莓派
- 树莓派开机

### 配置静态IP

本地登录centos

默认账号 root , 默认密码 centos

编写网卡配置，没有则创建

vi /etc/sysconfig/network-scripts/ifcfg-eth0

```
DEVICE=eth0
BOOTPROTO=static
IPADDR=192.168.0.103
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=192.168.0.1
ONBOOT=yes
PEERDNS=no
```

**注意：DNS1必须配置，树莓派重启后时间会回到1970年，如果dns服务器异常，会影响时间同步到正常时间**

重启网络

```
systemctl restart network
```

### 远程ssh 登录树莓派

网络配置好之后，就可以使用远程ssh工具 连接树莓派操作了，再下面的操作全部为远程ssh操作

mac 可以直接 ssh 连接， windows 可以使用putty 等工具连接 

### 优化ssh登录

编辑sshd_config，找到UseDNS，放开注释，配置为no

vi /etc/ssh/sshd_config

```
UseDNS no
```

重启sshd

```
systemctl restart sshd
```

### 禁用firewalld

```
systemctl disable firewalld --now
```

### 设置时区

```
timedatectl set-timezone Asia/Shanghai
```

### 优化sysctl

执行下面命令，在sysctl.conf 添加优化配置

```
cat >> /etc/sysctl.conf << EOF
kernel.unknown_nmi_panic=0
kernel.sysrq=1
kernel.pid_max=65535
net.ipv4.ip_forward=1
net.netfilter.nf_conntrack_max=1000000
fs.file-max=655350
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=10000000
net.core.wmem_max=16777216
net.core.rmem_max=16777216
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
fs.inotify.max_queued_events=327679
kernel.msgmnb=65536
kernel.msgmax=65536
kernel.shmmax=68719476736
kernel.shmall=4294967296
net.ipv4.neigh.default.gc_thresh1=2048
net.ipv4.neigh.default.gc_thresh2=4096
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv6.conf.all.disable_ipv6=1
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_tw_reuse=1
net.core.rmem_default=16777216
net.core.wmem_default=16777216
net.core.optmem_max=40960
net.ipv4.tcp_max_tw_buckets=2000000
net.ipv4.tcp_max_syn_backlog=30000
net.ipv4.tcp_max_orphans=262144
net.core.somaxconn=4096
net.core.netdev_max_backlog=50000
net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_syn_retries=2
net.ipv4.tcp_fin_timeout=7
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_keepalive_probes=5
net.ipv4.tcp_keepalive_intvl=3
net.ipv4.tcp_timestamps=0
net.ipv4.tcp_sack=1
net.ipv4.tcp_window_scaling=1
net.ipv4.tcp_abort_on_overflow=1
net.ipv4.ip_local_port_range=32768 60999
EOF

# 使sysctl.conf 生效
sysctl -p
```

### 扩展根目录磁盘

重新创建/dev/mmcblk0 第三分区

```
fdisk /dev/mmcblk0 < EOF

p
d
3
p
n
p
3
1593344

p
w
EOF
```

使分区生效

```
partprobe
resize2fs /dev/mmcblk0p3
```

### 格式化ssd扩展磁盘并挂载

ssd 磁盘创建分区

```
fdisk /dev/sda << EOF
n
p
1
2048
+100G

n
p
2


w
EOF
```

格式化ssd两个分区

```
mkfs.xfs /dev/sda1
mkfs.xfs /dev/sda2
```

创建/aos /data目录

```
mkdir -p /aos /data
```

编辑 /etc/fstab 文件

注释 swap 配置，添加新创建的磁盘分区配置到文件中，内容如下

vi /etc/fstab

```
UUID=f28e182b-76a9-4f22-9a79-d49691313906  / ext4    defaults,noatime 0 0
UUID=AF94-12D3  /boot vfat    defaults,noatime 0 0
#UUID=d61648c4-62a4-4ad7-84ca-4913ae013c7b  swap swap    defaults,noatime 0 0
/dev/sda1       /aos    xfs defaults,noatime 0 0
/dev/sda2       /data   xfs defaults,noatime 0 0
```

校验并挂载分区

```
mount -a
```

检查各挂载磁盘空间大小

```
df -h
```

### 禁用selinux

```
setenforce 0
```

编辑 /etc/selinux/config, 设置内容如下

```
SELINUX=permissive
```

### 重启

```
reboot
```