# 树莓派Raspberry Pi OS 设置静态ip


## 方法一（旧版本Raspberry Pi OS）

编辑 /etc/network/interfaces ，修改eth0 网卡相关配置如下：

```
source-directory /etc/network/interfaces.d
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.0.101
    netmask 255.255.255.0
    network 192.168.0.1
    gateway 192.168.0.1
 
```   
    
    
## 方法二（新版本Raspberry Pi OS）

编辑  /etc/dhcpcd.conf， 添加eth0 网卡相关配置如下：

```
interface eth0
static ip_address=192.168.0.103/24
static routers=192.168.0.103
```