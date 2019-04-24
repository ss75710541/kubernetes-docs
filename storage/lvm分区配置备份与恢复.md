# lvm分区配置备份与恢复测试

## 备份分区并下载到本地
```
vgcfgbackup -f vg_localvolume vg_localvolume
```

## 重装系统

## 安装 lvm

```
yum install lvm2
```
## 恢复分区配置

```
vgcfgrestore vg_localvolume -v -f vg_localvolume
```

## 创建对应挂载目录，添加到/etc/fstab, 激活分区

```
for i in {1..100};do
  mkdir -p /mnt/fast-disks/lv$i
  echo "/dev/mapper/vg_localvolume-lv$i /mnt/fast-disks/lv$i xfs defaults 0 0" >> /etc/fstab
  # 激活分区
  lvchange -ay /dev/mapper/vg_localvolume-lv$i
done
```

## 查看是否存在未激活分区

```
lvscan|grep -v ACTIVE
```

## 检查并挂载lvm

```
mount -a
```

## 查看挂载后的磁盘个数

```
df |grep vg_localvolume|wc -l
```

## 查看原有lvm分区数据

```
ls /mnt/fast-disks/lv1
```

