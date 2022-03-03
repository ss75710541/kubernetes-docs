# 使用image-syncer同步多CPU架构镜像到私有仓库

## 下载image-syncer

```
mkdir ~/image-syncers
cd ~/image-syncers
wget https://github.com/AliyunContainerService/image-syncer/releases/download/v1.3.1/image-syncer-v1.3.1-linux-amd64.tar.gz
tar xzvf image-syncer-v1.3.1-linux-amd64.tar.gz
```

## 创建 config.yaml

```yaml
auth:
  registry.hisun.netwarpps.com:
    username: "username"
    password: "xxxxxxxx"
  offlineregistry.example.com:5000:
    insecure: true
images:
  docker.io/library/ubuntu:21.10: registry.example.com/library/ubuntu:21.10
  registry.example.com/library/ubuntu:21.10: offlineregistry.example.com:5000/library/ubuntu:21.10
```

## 执行同步

```sh
./image-syncer --config config.yaml --arch "arm64" --arch "amd64" --os linux
```

## 验证其中一个镜像信息

```sh
docker manifest inspect registry.example.com/library/ubuntu:21.10

{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 529,
         "digest": "sha256:259c73507796f1175b4796cec9e1c783f2951af76ce7baeaec984892f5eee9e3",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 529,
         "digest": "sha256:f013b5ae7a5dfc9abc4d35bf9a3a2c18dd0c800226774ff4af05a364962a260d",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      }
   ]
}
```

## 参考

https://github.com/AliyunContainerService/image-syncer

https://docs.docker.com/engine/reference/commandline/manifest/