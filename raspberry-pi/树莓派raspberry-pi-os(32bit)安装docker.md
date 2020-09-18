# 树莓派raspberry-pi-os(32bit)安装docker

## 下载raspberry-pi-os镜像

下面链接是使用的清华源下载os镜像

```
wget https://mirrors.tuna.tsinghua.edu.cn/raspberry-pi-os-images/raspios_lite_armhf/images/raspios_lite_armhf-2020-08-24/2020-08-20-raspios-buster-armhf-lite.zip
```

下载完成可以去树莓派官网验证sha256

https://www.raspberrypi.org/downloads/raspberry-pi-os/

本文下载镜像 

SHA-256:4522df4a29f9aac4b0166fbfee9f599dab55a997c855702bfe35329c13334668

## 烧录镜像到SD/TF卡

使用 [balenaEtcher](https://www.balena.io/etcher/) 工具，把raspberry-pi-os 烧录到SD/TF卡

## 启用ssh

读卡器拔出再播放，显示 挂载为目录，进入目录创建 ssh 文件，内容不重要，有这个文件启动系统时就会开启ssh服务


## 安装docker

设置系统默认源为清华源

```
# 编辑 `/etc/apt/sources.list` 文件，删除原文件所有内容，用以下内容取代：
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib rpi
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib rpi

# 编辑 `/etc/apt/sources.list.d/raspi.list` 文件，删除原文件所有内容，用以下内容取代：
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

信任 Docker 的 GPG 公钥:

```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```

设置docker源为清华源

```
echo "deb [arch=armhf] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
```

```
apt update
apt install docker-ce docker-compose pass gnupg2
```

## 参考 

https://mirror.tuna.tsinghua.edu.cn/help/raspbian/

https://anto.online/guides/cannot-autolaunch-d-bus-without-x11-display/