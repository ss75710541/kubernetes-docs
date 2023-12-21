# OpenFaas使用Go模板创建Function

## 根据模板创建项目

```sh
# 登录openfaas
faas-cli login https://gateway-openfaas.netwarps.com -p xxxx
# 查看已经存在的function列表
faas-cli list
# 拉取语言模板
faas-cli template pull
# 根据默认go模板创建function项目
faas-cli new -p registry.solarfs.io/go-test  go-test2 --lang go
```

生成如下一个目录一个yml文件

```sh
go-test2
go-test2.yml
```

`go-test2.yml`如下

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: https://gateway-openfaas.netwarps.com
functions:
  go-test2:
    lang: go
    handler: ./go-test2
    image: registry.solarfs.io/go-test/go-test2:latest
    secrets: # 这个是手动添加的，配置镜像imagePullSecret
     - registry-pld-cicd # imagePullSecret name
```

构建镜像

```sh
faas-cli build -f ./go-test.yml --build-arg GOPROXY=https://proxy.golang.com.cn,direct
```

登录私有镜像仓库

```sh
faas-cli registry-login --username user --password pass
```

推送镜像

```sh
faas-cli push -f ./go-test.yml
```

部署function

```sh
faas-cli deploy -f ./go-test.yml
```

查看部署详细信息

```sh
faas-cli describe go-test
```

测试

```sh
echo "test go"|faas-cli invoke  go-test
```

删除function

```sh
faas-cli remove go-test
```

## 思考

### 减少代码量

openfaas 如果想要减少代码量，就是自定义提前定义代码模板, 提前把封装好的 模块 或  公共函数，比如说 web route logging metrics utils  等 写成模板，这样其它项目就可以从模板生成，直接写需要的业务代码了。也可以找别人开源的 代码模板，有现成的也可以用。

### 自动扩容

openfaas 开源版本扩容最多只能扩到5，再高的话，要买企业版，这个不实用, 不如使用k8s hpa 或 keda  实现自动扩容

## 参考

https://docs.openfaas.com/reference/yaml/

https://docs.openfaas.com/languages/go/