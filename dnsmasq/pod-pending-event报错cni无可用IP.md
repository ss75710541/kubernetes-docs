# 配置dnsmasq apps通配解析


配置通配域名

```
cat > /etc/dnsmasq.d/apps30-238.conf <EOF
address=/apps30-238.offline-okd.com/172.31.22.162
EOF
```

重启dnsmasq

```
systemctl restart dnsmasq
```

验证域名解析

```
ping grafana-openshift-monitoring.apps30-238.offline-okd.com
PING grafana-openshift-monitoring.apps30-238.offline-okd.com (172.31.22.162) 56(84) bytes of data.
64 bytes from infra22-162.offline-okd.com (172.31.22.162): icmp_seq=1 ttl=64 time=2.55 ms
64 bytes from infra22-162.offline-okd.com (172.31.22.162): icmp_seq=2 ttl=64 time=0.758 ms
```