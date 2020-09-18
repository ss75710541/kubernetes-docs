# 树莓派raspberry pi os开启ssh

## 在sd卡中写入ssh文件

ssh 文件内容不重要，生成一个文件就可以

```
cd /Volumes/boot
touch ssh
```

## 连接显示器、键盘在本地终端开启ssh

```
sudo systemctl enable ssh
sudo systemctl start ssh
```