# Mac OSX ssh隧道使用方法

## 配置ssh config

参考：[ssh_config 文件配置详解](http://blog.chinaunix.net/uid-8389195-id-1741604.html)

编辑 ~/.ssh/config，没有则新建，并修改权限

`touch ~/.ssh/config`

`chmod 600 ~/.ssh/config`

修改config文件的内容
   
``` 
ServerAliveInterval 60

Host *
   StrictHostKeyChecking no 	   # 忽略目标主机在本地key中不存储提示
    
Host *.workdomain.com            #hostname or ip
	IdentityFile ~/.ssh/id_rsa   #密钥所在地  
	User leo                     #用户名
	Port 22                      #端口号
 
Host zjkjump
	Hostname 1.2.3.4
 	User liujinye
	Port 22
	IdentityFile ~/.ssh/liujinye
```

登陆的时候，ssh会根据登陆不同的域来读取相应的私钥文件、端口、用户

## 建立隧道

### 方法一（ssh命令方法）

打开终端，然后直接用命令行

`ssh -N -p 22 -C -D 10020 liujinye@zjkjump -o StrictHostKeyChecking=no`

* -D 设置动态转发端口号；
* -C 启用压缩；
* -N 不执行远程shell命令(ssh2支持)，登录后不会有提示行；
* -i 优先使用秘钥key 而不是密码；

### 方法二（Mac GUI软件） 

[Mac OSX上的ssh tunnel 代理设置备忘](http://www.chedong.com/blog/archives/001403.html)

点击下载[ssh tunnel manager](http://www.macupdate.com/app/mac/10128/ssh-tunnel-manager)按照步骤安装完毕后启动添加跳板服务器通道设置。

***需要注意几个问题***

* 代理使用的是Socks,需要启用Socks4;
* 高级选项建议点选Handle authentication\Compress;
* 加密建议选择aes256-cbc;
* 建议不选择auto connecet否则启动就会自动连接;

#### 关于多私钥问题
[ssh tunnel manager](http://www.macupdate.com/app/mac/10128/ssh-tunnel-manager)不支持密钥选择，如果多密钥或者密钥不在默认.ssh目录下此程序将无法正常运转。

## ssh登录内部主机

编辑 ~/.ssh/config 添加ssh proxy 代理

```
Host 172.26.*.*
	User root
	Port 22
	ProxyCommand nc -x 127.0.0.1:10020 %h %p
```

ssh 172.26.163.181 可以直接通过隧道10020端口直接登录到内网主机

## web访问内部主机

使用chrome下载 [SwitchyOmega](https://proxy-switchyomega.com
) 插件， 
安装后添加 情景模式 代理协议使用socks5 (使用隧道的dnsserver,socks4使用本地dnsserver), 代理服务器填写 `127.0.0.1`,代理端口 `10020`

访问内部web页面时，需要选择相应的情景模式，或创建自动切换策略

参考：[SwitchyOmega配置](https://proxy-switchyomega.com/settings/)
