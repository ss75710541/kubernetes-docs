# 树莓派4B+raspberry-pi-os-buster在线安装k3s

## 安装docker

略

## 在 Raspbian Buster 上启用旧版的 iptables
Raspbian Buster 默认使用nftables而不是iptables。 K3S 网络功能需要使用iptables，而不能使用nftables。 按照以下步骤切换配置Buster使用legacy iptables：

```
sudo iptables -F
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo reboot
```

```
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh - --docker --node-name raspberrypi
```

密码列表

```
/var/lib/rancher/k3s/server/cred/passwd 
```

## 安装dashboard

略

### 获取dashboard 管理token

sudo k3s kubectl -n kubernetes-dashboard describe secret admin-user-token | grep token

##

kubectl exec -it nginx-64bc6d46b9-p8qr4 -- sh

## 参考

https://docs.rancher.cn/docs/k3s/advanced/_index/#%E5%9C%A8-raspbian-buster-%E4%B8%8A%E5%90%AF%E7%94%A8%E6%97%A7%E7%89%88%E7%9A%84-iptables

https://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index

https://docs.rancher.cn/docs/k3s/installation/install-options/_index
