# ucloud环境开启selinux


## 问题描述

ucloud 能centos7.6 镜像创建主机默认不开启selinux, 按常规方法配置，开启selinux后，主机ping 不能，web console 也不能正常登录

/etc/selinux/config文件

```
SELINUX=permissive
SELINUXTYPE=targeted
```


## 解决方法

### 1.新建主机后 修改/etc/selinux/config文件，selinux设置为 permissive (宽容模式)

```
SELINUX=permissive
SELINUXTYPE=targeted
```
### 2.重启虚机
```
reboot
```
### 3.使用以下命令reset权限位
```
restorecon -R -v /etc
restorecon -R -v /usr
```
### 4.再次重启虚机
```
reboot
```
### 5.selinux 改为 enforcing (强制模式)

修改/etc/selinux/config文件, 

```
SELINUX=enforcing
```

### 6.再次重启虚机

```
reboot
```

### 7. 现在已经是正常开始了selinux, 并且所有功能正常，此状态下的主机镜像可以制作为 自制镜像，以后使用自制镜像创建主机，默认就是开启selinux状态了。