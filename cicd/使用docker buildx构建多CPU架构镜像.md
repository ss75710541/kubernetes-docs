# 使用 docker buildx 构建多CPU架构镜像

| 环境组件          | 版本            |
| ----------------- | --------------- |
| 操作系统          | Rocky Linux 8.5 |
| CPU 架构          | x86_64          |
| Docker            | 20.10.12        |
| Harbor            | v2.3.2          |
| Kubernetes (可选) | 1.22.2          |

## 创建多平台构建器

```sh
docker buildx create --name multi-platform --use --platform linux/amd64,linux/arm64 --driver docker-container
```

## 创建单机容器多平台构建器(与k8s构建器二选一)

### 启用多CPU架构静态编译环境

```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

### 创建单机窗口多平台构建器并默认使用

```
docker buildx create --use --name mybuilder
```

### 启动并查看构建器

```sh
docker buildx inspect mybuilder --bootstrap

Name:   mybuilder
Driver: docker-container

Nodes:
Name:      mybuilder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64, linux/arm64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6, linux/riscv64, linux/mips64le, linux/mips64
```

### 创建k8s多平台构建器(与单机容器构建器二选一)

#### 准备k8s环境

#### 登录master主机创建k8s多平台构建器并默认使用

```bash
kubectl create ns docker-builder
docker buildx create --use --name mutil-arth-k8s --driver kubernetes --driver-opt nodeselector=kubernetes.io/arch=amd64 --driver-opt replicas=1 --driver-opt qemu.install=true --driver-opt namespace=docker-builder
```

#### 启动并查看构建器

```
docker buildx inspect mutil-arth-k8s --bootstrap

Name:   mutil-arth-k8s
Driver: kubernetes

Nodes:
Name:      mutil-arth-k8s0-76b5c6d889-47qbg
Endpoint:
Status:    running
Platforms: linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```

## 构建多平台构建并推送到仓库

### 准备 Dockerfile

```dockerfile
FROM golang:1.17-alpine as builder

ENV GO111MODULE=on
ENV GOPROXY=https://goproxy.io

COPY rnode-exporter/. ./rnode-exporter/
RUN cd ./rnode-exporter/ && \
	go mod download && \
	CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o rnode-exporter && \
	cp rnode-exporter /

FROM alpine:3.15 as prod

COPY --from=0 rnode-exporter /

EXPOSE 9102

CMD ["/rnode-exporter"]
```

### 登录镜像仓库

注意：harbor 2.x 以后才支持多CPU架构构建共存

```
docker login harbor.example.com
```

### 执行构建并推送到仓库

```sh
docker buildx build  -t harbor.example.com/bfs/rnode-exporter:v0.10.3 --platform=linux/arm64,linux/amd64 . --push
```

参考：

https://docs.docker.com/engine/reference/commandline/buildx_create/

https://github.com/docker/buildx/issues/495