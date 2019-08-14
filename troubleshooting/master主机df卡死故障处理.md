# master 主机df 卡死

df 卡死多数 是因为挂载 了远程网盘，连接远程网盘异常导致的

查询挂载的nfs

mount|grep nfs 

卸载已经挂载的nfs

umount 172.30.31.145:/export/pvc-0315276d-300a-11e9-9147-00163e02b8be

df 正常
