# MAC 环境使用 dnsmasq 配置openshift相关自定义域名


## 安装 `dnsmasq`

```
brew install dnsmasq
```

## copy 主配置

```
cp /usr/local/opt/dnsmasq/dnsmasq.conf.example /usr/local/etc/dnsmasq.conf
```

## 添加引用外部dns的配置

echo 'server=114.114.114.114' > /usr/local/etc/dnsmasq.d/dns.conf
ecoh 'server=8.8.8.8' >> /usr/local/etc/dnsmasq.d/dns.conf

## 添加自定义通配域名解析

x.x.x.x 为 openshift router ip 

```
echo 'address=/apps.offline-okd.com/x.x.x.x' >> /usr/local/etc/dnsmasq.d/address.conf
```

## 重启 `dnsmasq`

```
sudo launchctl stop homebrew.mxcl.dnsmasq
sudo launchctl start homebrew.mxcl.dnsmasq
sudo killall -HUP mDNSResponder
```