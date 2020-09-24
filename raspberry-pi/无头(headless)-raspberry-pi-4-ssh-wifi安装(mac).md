# 无头(headless) raspberry pi 4 ssh wifi 安装(mac)

## 步骤1.下载Raspberry Pi OS（以前称为Raspbian）Buster lite

从下面地址下载Raspberry Pi OS Buster镜像：

https://www.raspberrypi.org/downloads/raspberry-pi-os/

我下载使用的是 2020年8月20日发布的lite image（无桌面）内核5.4版。

## 步骤2.将Raspberry Pi OS镜像刻录到SD卡

使用 Etcher 将 镜像记录到SD卡

- 下载地址 https://www.balena.io/etcher/
- 下载适用操作系统的版本etcher
- 运行安装程序

运行Etcher非常简单

将空白的微型SD卡和读卡器插入电脑。无需格式化。可以直接使用新的SD卡。

1. 单击`select image` 选择下载好的Raspberry Pi OS zip文件。
2. 单击`Select target` 一般会自动找到SD卡，如果没有选择对应SD卡
3. 单击`Flash!` 可能会提示您输入当前使用系统密码

在你flash(刻录)图像后，Finder (Mac)或文件资源管理器(Windows)可能很难看到它。一个简单的解决方法是取出SD卡，然后再把它插回去。在Mac上，它应该以boot的名称出现在桌面上。

## 步骤3.启用ssh以允许远程登录

出于安全原因，默认情况下不再启用ssh。要启用它，需要在启动磁盘的根目录中放置一个名为ssh（无扩展名）的空文件。

### 开启ssh

打开一个终端窗口并运行以下命令：

```
touch /Volumes/boot/ssh
```

## 步骤4.添加WiFi网络信息

在启动的根目录中创建一个名为wpa_supplicant.conf的文件（以下说明）。然后将以下内容粘贴到其中（调整您的[ISO 3166 alpha-2国家/地区代码](https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes)，网络名称和网络密码）：

创建一个新的wpa_supplicant.conf空文件

```
touch /Volumes/boot/wpa_supplicant.conf
```

复制下面内容到 wpa_supplicant.conf ，根据需要修改 国家代码、 wifi 名称和密码

```
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="NETWORK-NAME"
    psk="NETWORK-PASSWORD"
}
```

## 步骤5.弹出微型SD卡

右键单击启动（在桌面或文件资源管理器上），然后选择“弹出”选项

## 步骤6.从Micro SD卡启动Raspberry Pi

SD卡插入树莓派，插好电源线，打开电源开关，给Pi足够的时间来启动（可能需要90秒或更长时间）

## 步骤7.通过WiFi远程登录

登录路由器 查看最新连接的 主机名为 raspberrypi 的设置ip

raspberry pi 默认用户是pi，密码为raspberry。

```
ssh pi@myip
```

## 步骤8.更改主机名和密码

```
sudo raspi-config
```

## 步骤9.获取最新更新

```
sudo apt-get update -y
sudo apt-get upgrade -y
```

## 步骤10.配置无线静态IP

配置静态ip，编辑`/etc/dhcpcd.conf` 

```
sudo vi /etc/dhcpcd.conf
```

添加以下内容(无线为wlan0网卡，默认有线为eth0)：

```
interface wlan0
static ip_address=192.168.0.105
static routers=192.168.0.1
```

要确认设置为永久设置，请重启

```
sudo reboot
```

```
ssh pi@1192.168.0.105
```

## 故障排除

以下是一些有关在Pi上调试网络和wifi问题的有用命令：

- 该命令应在wlan0的第一行中列出您的网络：

```
iwconfig
```

- 此命令应显示wlan0的信息

```
ifconfig
```

- 此命令应列出您的网络名称

```
iwlist wlan0 scan | grep ESSID
```

- 要编辑或查看您的wifi设置，请运行以下命令

```
sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
```

- 要在编辑配置文件后加载（可能需要重新登录）：

```
sudo wpa_cli -i wlan0 reconfigure
```

## 参考

https://desertbot.io/blog/headless-raspberry-pi-4-ssh-wifi-setup