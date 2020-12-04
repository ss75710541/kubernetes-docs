# k8s如果防止容器中出现僵尸进程

## 1、直接运行服务进程

Dockerfile 如下：

```
FROM alpine:3.12
COPY server /
EXPOSE 8080
ENTRYPOINT ["/server"]
CMD ["arg1"]
```

这种方式启动的服务,server 在容器中的进程pid是1，主进程server被kill, 会重建容器，旧容器被杀死后的残留进程被宿主机的1号进程回收残留进程

## 2、bash 使用 exec 运行进程

服务的启动脚本和 Dockerfile 示例如下：

run.sh

```
#!/bin/bash
mkdir -p /var/log/server
exec /server
```
Dockerfile

```
FROM alpine:3.12
COPY server run.sh /
EXPOSE 8080
ENTRYPOINT ["/run.sh"]
CMD ["args1"]
```

如果需要在启动前做一些脚本化的操作，再启动服务，这种是在`run.sh`最后启动服务的时候， 使用exec server, 会让server进程替换当前shell 做1号进程，这种情况下同 1直接运行服务进程，也就不会产生多余全局进程。

## 3、pod使用 hostPID

如果一个容器中有嵌套守护进程的逻辑，容器中的1号守护进程启动子进程，子进程还会启动其它子进程，中间的子进程被Kill 还是会出现全局进程的情况

如下所示，1号进程是 `looprun-x64`, 子进程是 `./9234`, `[9234] <defunct>` 就是中间进程被kill 掉之后，父进程自动改为1， 但是容器中的守护进程服务 `looprun-x64` 又没有回收僵尸进程的能力，就会产生大量僵尸进程.


``` ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
ks           1     0  0 Nov13 ?        00:00:22 ./looprun-x64
ks          54     1  0 Nov13 ?        00:00:00 [9234] <defunct>
ks         130     1  0 Nov13 ?        00:00:08 [9234] <defunct>
ks        2385     0  0 14:46 ?        00:00:00 /bin/sh
ks        7446     1  0 Nov15 ?        00:00:11 [9234] <defunct>
ks       15315     1  0 Nov23 ?        00:00:06 [9234] <defunct>
ks       15800     1  0 12:31 ?        00:00:00 [9234] <defunct>
...
ks       15879     1  0 12:32 ?        00:00:06 ./9234
ks       16017     1  0 Nov25 ?        00:00:29 [9234] <defunct>
```

这种情况解决方法之一就是容器的进程pid 与宿主机共享，让宿主机的1号进程`/sbin/init` 来回收僵尸进程

k8s 的发现在yaml 的参数中添加 `hostPID: true`

```
...
spec:
  containers:
	...
  hostPID: true
  ...
```

容器启用宿主的pid 之后，容器中的主进程id 就不是1了，就算中间进程 `./9234` 被kill 其下的子进程会自动设置宿主机的 1号进程为父id, 宿主机的 1号进程 `/sbin/init` 是有回收无用的僵尸进程能力的，这相也就不会有多余的僵尸进程。

### 理解进程主机共享

1. 容器进程不再具有 PID 1。
2. 进程对宿主机中的其它进程可见。
3. 容器文件系统通过 /proc/$pid/root 链接对 pod 中的其他容器可见。 这使调试更加容易，但也意味着文件系统安全性只受文件系统权限的保护。

## 4、docker 启动命令使用Tnin

上述三种情况都是因为容器中的1号进程不具备回收子进程的能力导致的，所以如果我们让 容器中的 1号进程拥有回收子进程的能力，也就不会有僵尸进程。

Tini 做为1号进程启动，Tini 有进程管理的功能，可以回收僵尸进程，也不会对容器暴露主机上的进程，在旧版本的k8s中，这个是个不错的解决方案，很多大型项目也都有使用。

Tnin github 地址：https://github.com/krallin/tini

dockerfile 示例如下：

```
# Add Tini
ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
ENTRYPOINT ["/tini", "--"]

# Run your program under Tini
CMD ["/your/program", "-and", "-its", "arguments"]
# or docker run your-image /your/program ...
```

## 5、pod使用 `shareProcessNamespace: true`

从 k8s v1.17+ 起 ，`ShareProcessNamespace` 字段 就已经是 stable 状态了，启用后pod 中多个容器间可以共享进程，Infra容器中启动一个pause的进程作为1号进程管理pod 中的子进程，这样就可以回收容器中的僵尸进程了。

### 理解进程命名空间共享
 
1. 容器进程不再具有 PID 1。 在没有 PID 1 的情况下，一些容器镜像拒绝启动（例如，使用 systemd 的容器)，或者拒绝执行 kill -HUP 1 之类的命令来通知容器进程。在具有共享进程命名空间的 pod 中，kill -HUP 1 将通知 pod 沙箱（在上面的例子中是 /pause）。
2. 进程对 pod 中的其他容器可见。 这包括 /proc 中可见的所有信息，例如作为参数或环境变量传递的密码。这些仅受常规 Unix 权限的保护。
3. 容器文件系统通过 /proc/$pid/root 链接对 pod 中的其他容器可见。 这使调试更加容易，但也意味着文件系统安全性只受文件系统权限的保护。

### k8s 中发布参考配置

```
...
spec:
  shareProcessNamespace: true
  containers:
	...
```



参考：https://www.cnblogs.com/kkbill/p/12952815.html
参考：https://blog.csdn.net/whatday/article/details/104136230