# ucloud环境开启selinux后/var/log/messages不能写入问题

`journalctl -f` 中报 /var/log/messages 没有权限错误

```
Sep 03 10:46:09 fnode172-16-121-178.solarfs.okd rsyslogd[1438]: file '/var/log/messages': open error: Permission denied [v8.24.0-52.el7_8.2 try http://www.rsyslog.com/e/2433 ]
```

查看selinux 上下文 label

```
ls -Z /var/log/messages

-rw-------. root   root            system_u:object_r:unlabeled_t:s0 /var/log/messages
```

去掉/var/log/messages不可写入的权限 

```
chattr -ai /var/log/messages
```

修改/var/log/messages 上下文 label

```
semanage fcontext -a -t var_log_t /var/log/messages
```

恢复为正确安全上下文

```
restorecon -R -v /var/log/messages
```

查看/var/log/messages 安全上下文

```
ls -Z /var/log/messages


-rw-------. root root system_u:object_r:var_log_t:s0   /var/log/messages
```

查看 /var/log/messages 有新日志生成

```
tail -f /var/log/messages
```

导致这个原因的初始原因是在 ucloud 开启selinux 时没有恢复 /var目录安装上下文件导致的，如果最开始，直接 `restorecon -R -v /`, 不会有这个问题。