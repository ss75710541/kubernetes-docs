# OpenShift Docker Registry 500

OpenShift 上 Build Image 成功，但 Push 时 Docker Registry 出现 500 错误。

```
Pushing image docker-registry.default.svc:5000/xxxxx/xxx:latest ...
Pushed 6/9 layers, 67% complete
Pushed 7/10 layers, 78% complete
Pushed 8/9 layers, 89% complete
Registry server Address: 
Registry server User Name: serviceaccount
Registry server Email: serviceaccount@example.org
Registry server Password: <<non-empty>>
error: build error: Failed to push image: received unexpected HTTP status: 500 Internal Server Error
```

检查 registry pod log发面下面日志

```
....
time="2019-07-24T09:44:04.766206734Z" level=error msg="response completed with error" err.code=UNKNOWN err.detail="filesystem: mkdir /registry/docker/registry/v2/repositories/xxxxx: permission denied" err.message="unknown error" go.version=go1.12.7 http.request.host="172.30.91.235:5000" .... http.response.status=500 ....
....
```

进入registry 容器中查看registry 数据目录，发现数据目录属主为 `root:1000000000`，但registry 启动用户为 1001,

registry 后端存储使用了ceph

由于前一阵子修改了pod 默认的启动用户，registry 之前 创建的数据目录权限 为 `root:1000000000`,registry pod 挂掉重启之后，用户变更为1001所以没有了写入权限。

openshift cli 不支持切换用户进入容器，所以需要找到registry pod 对应的主机容器,以root用户进入容器内部

```
docker exec -it -u root k8s_registry_docker-registry-1-h7vln_default_734aa8f3-1fb7-11e9-bb46-a02bb81f82fc_0 bash
chown -R 1001:1001 /registry/
```

改完属主权限后，重新build image ，push正常了
