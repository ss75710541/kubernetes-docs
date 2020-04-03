# 使用alpine-make-vm-image制作alpine-qcow2云镜像


参考 ：

https://curlybracket.co.uk/blog/running-alpine-linux-on-digital-ocean/
https://www.tianjiaguo.com/2018/03/qcow2-cloud-image-custom-guide/
http://zvampirem.com/install-nbd-in-centos/


## 环境 cetnos 7.4  升级到 7.7



```
yum update -y
reboot
```


## 安装qeum-img

```
yum install -y qemu-img
```

## 安装编译工具

```
yum install -y make gcc
```

## 编译安装ndb组件 

```
yum install -y kernel-devel  elfutils-libelf-devel
# 下载对应的内核源码包
wget https://mirrors.aliyun.com/centos-vault/7.7.1908/os/Source/SPackages/kernel-3.10.0-1062.el7.src.rpm
rpm -ihv kernel-3.10.0-1062.el7.src.rpm
cd /root/rpmbuild/SOURCES/
tar xvf linux-3.10.0-1062.el7.tar.xz -C /usr/src/kernels/
cd /usr/src/kernels/
# 配置源码
mv $(uname -r) $(uname -r)-old
mv linux-3.10.0-1062.el7 $(uname -r)
# 编译安装
cd $(uname -r)
make mrproper
cp ../$(uname -r)-old/Module.symvers ./
cp /boot/config-$(uname -r) ./.config
make oldconfig
make prepare
make scripts
sed -i 's/REQ_TYPE_SPECIAL/7/g' drivers/block/nbd.c
make CONFIG_BLK_DEV_NBD=m M=drivers/block
cp drivers/block/nbd.ko /lib/modules/$(uname -r)/kernel/drivers/block/
depmod -a
# 查看nbd模块
modinfo nbd
# 重启主机
reboot
```

## 准备alpine-make-vm-image

```
mkdir alpine
cd alpine
wget https://raw.githubusercontent.com/alpinelinux/alpine-make-vm-image/master/alpine-make-vm-image && chmod +x alpine-make-vm-image
wget https://raw.githubusercontent.com/alpinelinux/alpine-make-vm-image/master/example/configure.sh && chmod +x configure.sh
```

```
cat > repositories <<EOF
http://mirror.lzu.edu.cn/alpine/v3.11/main
http://mirror.lzu.edu.cn/alpine/v3.11/community
EOF
```

## 制作alpine 镜像

制作镜像

```
./alpine-make-vm-image \
  --image-format qcow2 \
  --repositories-file ./repositories \
  --mirror-uri http://mirror.lzu.edu.cn/alpine \
  --image-size 1G \
  --packages "ca-certificates ssl_client chrony less logrotate openssh sudo" \
  --script-chroot \
  alpine-v3.11-$(date +%Y-%m-%d).qcow2 -- ./configure.sh
```

压缩镜像

```
gzip alpine-v3.11-$(date +%Y-%m-%d).qcow2 
```