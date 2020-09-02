# ucloud 添加负载均衡报文转发配置

ucloud环境添加 报文转发型 负载均衡，需要把 外部访问的 外网IP 添加在后端真实服务节点上, CentOS 环境配置如下：

注意：如果是CentOS 8.x 需要先安装 network-scripts

```
yum install network-scripts:
```

```
VIP=101.36.118.34

cat > /etc/sysconfig/network-scripts/ifcfg-lo:1 <<EOF
DEVICE=lo:1
IPADDR=$VIP
NETMASK=255.255.255.255
EOF

ifup lo:1
```



参考： https://docs.ucloud.cn/ulb/guide/realserver/editrealserver