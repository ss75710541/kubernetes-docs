# Hyperledger Fabric on openshift 3.11


## 添加3个openshift node 节点

登录管理员账号

```
oc login
```

给新加节点添加`label` `fabric=true`

```
for i in 82 83 84
do
	oc label node node$i.dmos.dataman fabric=true
done
```

## 安装 fabric 二进制

```
wget https://github.com/hyperledger/fabric/releases/download/v2.0.1/hyperledger-fabric-linux-amd64-2.0.1.tar.gz
tar -xzf hyperledger-fabric-linux-amd64-2.0.1.tar.gz
# 移动 bin 目录二进制
mv bin/* /bin
# 检查是否安装成功
configtxgen --version
# configtxgen:
#  Version: 2.0.1
#  Commit SHA: 1cfa5da98
#  Go version: go1.13.4
#  OS/Arch: linux/amd64
```

## 下载fabric-external-chaincodes

```
git clone https://github.com/ss75710541/fabric-external-chaincodes.git
```

## 初始化配置


`configtx.yaml` 和 `crypto-config.yaml` 已经配置好3 个 RAFT orderer 服务，2个组织，每个组织有一个peer(org1和org2)，每个组织有一个CA。默认不需要修改这些文件。

生成所有密钥文件、channel的创世块、CA证书文件使用下面脚本

```
cd fabric-external-chaincodes
chmod +x fabricOps.sh
./fabricOps.sh start
```

## 同步 fabric-external-chaincodes 目录到fabric 节点

在/etc/ansible/hosts中添加fabric 组

```
[fabric]
node82.dmos.dataman
node83.dmos.dataman
node84.dmos.dataman
```
同步目录，设置buildpack/bin 脚本执行权限

```
ansible fabric -m copy -a "src=/root/fabric/fabric-external-chaincodes/ dest=/home/fabric-external-chaincodes/"
ansible fabric -m shell -a "chmod 755 /home/fabric-external-chaincodes/buildpack/bin/*"
```

## 创建project

```
kubectl create ns hyperledger
```

## 部署 orderer-service

```
oc create sa fabric
oc adm policy add-scc-to-user privileged system:serviceaccount:hyperledger:fabric
```

分别在orderer-service/orderer[0-2]-deployment.yaml 文件中添加节点选择亲和性
```
...
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: fabric
                operator: In
                values:
                - 'true'
...
      serviceAccountName: fabric
...
```
发布orderer-service

```
kubectl create -f orderer-service/
```

检查服务启动状态

```
kubectl get pods -n hyperledger
```

## 部署 org1 （组织1）

```
kubectl create -f org1/
```

检查服务状态

```
kubectl get pods -n hyperledger
```

## 部署 org2 （组织2）

```
kubectl create -f org2/
```

检查服务状态

```
kubectl get pods -n hyperledger
```

## 设置 channels（通道）

进入cli-org1 容器中

```
oc -n hyperledger rsh cli-org1-57b7d7d848-fl7f7
```
创建 channel

```
peer channel create -o orderer0:7050 -c mychannel -f ./scripts/channel-artifacts/channel.tx --tls true --cafile $ORDERER_CA
```

org1 从创建块 加入 mychannel

```
peer channel join -b mychannel.block
```

进入cli-org2 容器中

```
oc exec -it cli-org2-58658b9c96-nchbg sh -n hyperledger

```

获取 mychannel 

```
peer channel fetch 0 mychannel.block -c mychannel -o orderer0:7050 --tls --cafile $ORDERER_CA
```

org2 从创建块 加入 mychannel

```
peer channel join -b mychannel.block
```

查看 当前 peer 加入的 channel 列表

```
peer channel list
```

## Peer节点安装外部 Chaincode（链码）

### 配置连接 chaincode 信息

进入 cli-org1

```
oc -n hyperledger rsh cli-org1-57b7d7d848-fl7f7
```

打包链码连接信息

```
cd /opt/gopath/src/github.com/marbles/packaging
tar cfz code.tar.gz connection.json
tar cfz marbles-org1.tar.gz code.tar.gz metadata.json
peer lifecycle chaincode install marbles-org1.tar.gz

### 显示如下
2020-04-16 09:35:49.427 UTC [cli.lifecycle.chaincode] submitInstallProposal -> INFO 001 Installed remotely: response:<status:200 payload:"\nHmarbles:3ec68d1efc8a0e69beec80e8cba0087cb183edf25372202c5bf099b1e4c69ded\022\007marbles" >
2020-04-16 09:35:49.427 UTC [cli.lifecycle.chaincode] submitInstallProposal -> INFO 002 Chaincode code package identifier: marbles:3ec68d1efc8a0e69beec80e8cba0087cb183edf25372202c5bf099b1e4c69ded
```

记下上面的 链码包标识符, 后面会使用，如果没记住，可以使用下面命令查询 链码包标识符

```
peer lifecycle chaincode queryinstalled

### 显示如下
Installed chaincodes on peer:
Package ID: marbles:3ec68d1efc8a0e69beec80e8cba0087cb183edf25372202c5bf099b1e4c69ded, Label: marbles
```

现在我们为org2重复上面的步骤，但是由于我们希望用另一个pod为org2的peer节点提供链码服务， 因此我们需要修改connection.json中的地址配置：

```
"address": "chaincode-marbles-org2.hyperledger:7052",
```

进入 cli-org2

```
oc -n hyperledger rsh cli-org2-58658b9c96-nchbg
cd /opt/gopath/src/github.com/marbles/packaging
sed -i 's/chaincode-marbles-org1.hyperledger/chaincode-marbles-org2.hyperledger/g' connection.json
rm -f code.tar.gz
tar cfz code.tar.gz connection.json
tar cfz marbles-org2.tar.gz code.tar.gz metadata.json
peer lifecycle chaincode install marbles-org2.tar.gz

### 显示如下
2020-04-16 09:37:52.749 UTC [cli.lifecycle.chaincode] submitInstallProposal -> INFO 001 Installed remotely: response:<status:200 payload:"\nHmarbles:1b05e4523b4dd26ffbcd884a856eb58fab6f76591e7c5b00825fe2382e9ed822\022\007marbles" >
2020-04-16 09:37:52.750 UTC [cli.lifecycle.chaincode] submitInstallProposal -> INFO 002 Chaincode code package identifier: marbles:1b05e4523b4dd26ffbcd884a856eb58fab6f76591e7c5b00825fe2382e9ed822
```
同样记录链码包的标识符，它应该与之前org1的不同。

## Fabric外部链码的构建与部署

### 制作 chaincode docker 镜像

因为部署openshift 3.11 使用的docker版本为1.13.x, 不支持多层dockerfile 构建 ，所以在hub.docker.com 创建了自动根据 [Dockerfile](https://github.com/ss75710541/fabric-external-chaincodes/tree/master/chaincode/Dockerfile) 打镜像的repo

没有修改需求的可以直接使用 `ss75710541/chaincode-marbles:1.0` 镜像

### 部署chaincode

```
cd chaincode
```

修改 `k8s/org1-chaincode-deployment.yaml` 中下面内容

```
      containers:
        - image: ss75710541/chaincode-marbles:1.0
          env
            - name: CHAINCODE_CCID
              value: "marbles:3ec68d1efc8a0e69beec80e8cba0087cb183edf25372202c5bf099b1e4c69ded"
```


修改 `k8s/org2-chaincode-deployment.yaml` 中下面内容

```
      containers:
        - image: ss75710541/chaincode-marbles:1.0
          env
            - name: CHAINCODE_CCID
              value: "marbles:1b05e4523b4dd26ffbcd884a856eb58fab6f76591e7c5b00825fe2382e9ed822"
```

```
oc create -f k8s/
```

观察 pod 状态

```
oc get pod

### 显示如下
NAME                                      READY     STATUS    RESTARTS   AGE
ca-org1-5d578756d5-r5xsr                  1/1       Running   0          6m
ca-org2-7d97cc4775-j2mz8                  1/1       Running   0          6m
chaincode-marbles-org1-7496769f86-gqsw2   1/1       Running   0          4s
chaincode-marbles-org2-845c58594f-hmr5h   1/1       Running   0          4s
cli-org1-57b7d7d848-fl7f7                 1/1       Running   0          6m
cli-org2-58658b9c96-nchbg                 1/1       Running   0          6m
orderer0-7d79f689d6-cjqwl                 1/1       Running   0          6m
orderer1-7886dfffd5-rbm4n                 1/1       Running   0          6m
orderer2-7fb7865fcd-tcfwg                 1/1       Running   0          6m
peer0-org1-85c46dcbf4-ts9qf               1/1       Running   0          6m
peer0-org2-898cb5fdc-82d6w                1/1       Running   0          6m
```

进入cli-org1 审批org1 chaincode 定义, 注意修改命令中的`--package-id`值

```
oc rsh cli-org1-57b7d7d848-fl7f7
peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:3ec68d1efc8a0e69beec80e8cba0087cb183edf25372202c5bf099b1e4c69ded --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

### 显示如下
2020-04-16 09:26:22.781 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [9922dd879b5796113339f8b876eab17e3066b02eb450e52debca6073a7d392c1] committed with status (VALID) at
```

进入cli-org2 审批org2 chaincode 定义, 注意修改命令中的`--package-id`值

```
oc rsh cli-org2-58658b9c96-nchbg
peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:1b05e4523b4dd26ffbcd884a856eb58fab6f76591e7c5b00825fe2382e9ed822 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

### 显示如下
2020-04-16 09:30:00.446 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [145bd306c74d21a170361c6bee4df85c727676b069d254d4e18ae91a04864661] committed with status (VALID) at
```

检查chaincode 提交准备状态

```
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-required --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

# 显示如下
Chaincode definition for chaincode 'marbles', version '1.0', sequence '1' on channel 'mychannel' approval status by org:
org1MSP: true
org2MSP: true
```

在通道中提交这个链码的定义（可以在任意peer cli）

```
peer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name marbles --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

# 显示如下
2020-04-16 10:01:49.800 UTC [chaincodeCmd] ClientWait -> INFO 002 txid [331dacfb44c6c52d39e4b91566c6569efa921b4153e05a655a630a4bb070e451] committed with status (VALID) at peer0-org1:7051
2020-04-16 10:01:49.800 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [331dacfb44c6c52d39e4b91566c6569efa921b4153e05a655a630a4bb070e451] committed with status (VALID) at peer0-org2:7051
```

查询已经提交了的 链码定义 

```
peer lifecycle chaincode querycommitted -o orderer0:7050  --channelID mychannel  --name marbles

# 显示如下
Committed chaincode definition for chaincode 'marbles' on channel 'mychannel':
Version: 1.0, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc, Approvals: [org1MSP: true, org2MSP: true]
```

## 测试外部chaincode

我们可以从cli pod中测试链码的查询和交易调用。尝试创建一个宝石并初始化链码：

```
peer chaincode invoke -o orderer0:7050 --isInit --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble1","blue","35","tom"]}' --waitForEvent

# 显示如下
2020-04-16 10:22:22.066 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [926ab8f5aa67163d368aa7ad0a9b4e0c433c66115742fba890ff080777528d0d] committed with status (VALID) at peer0-org1:7051
2020-04-16 10:22:22.067 UTC [chaincodeCmd] ClientWait -> INFO 002 txid [926ab8f5aa67163d368aa7ad0a9b4e0c433c66115742fba890ff080777528d0d] committed with status (VALID) at peer0-org2:7051
2020-04-16 10:22:22.067 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
```

查询 创建的宝石 marble1，报错提示不存在

```
peer chaincode query -C mychannel -n marbles -c '{"Args":["readMarble","marble1"]}'

# 显示如下
Error: endorsement failure during query. response: status:500 message:"{\"Error\":\"Marble does not exist: marble1\"}"
```

可能是因为上面创建宝石命令使用了 `--isInit` 只是初始化了链码，但是没有真正创建宝石marble1，去掉 `--isInit`, 再创建一次

```
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble1","blue","35","tom"]}' --waitForEvent

# 显示如下
2020-04-16 10:32:59.934 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [be1bbd459d2ff1263eb6f03142907ec65be9ff7c935f9433c386b19426eab48e] committed with status (VALID) at peer0-org1:7051
2020-04-16 10:32:59.935 UTC [chaincodeCmd] ClientWait -> INFO 002 txid [be1bbd459d2ff1263eb6f03142907ec65be9ff7c935f9433c386b19426eab48e] committed with status (VALID) at peer0-org2:7051
2020-04-16 10:32:59.935 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
```

查询宝石信息：

```
peer chaincode query -C mychannel -n marbles -c '{"Args":["readMarble","marble1"]}'

# 显示如下
{"docType":"marble","name":"marble1","color":"blue","size":35,"owner":"tom"}
```

创建另一个宝石：

```
peer chaincode invoke -o orderer0:7050  --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble2","red","50","tom"]}' --waitForEvent

# 显示如下
2020-04-16 10:23:01.838 UTC [chaincodeCmd] ClientWait -> INFO 001 txid [1bf9a8cf6331282ee44b8e4a61d2f53d9a7e2dea437942477a93797b021d1334] committed with status (VALID) at peer0-org1:7051
2020-04-16 10:23:01.839 UTC [chaincodeCmd] ClientWait -> INFO 002 txid [1bf9a8cf6331282ee44b8e4a61d2f53d9a7e2dea437942477a93797b021d1334] committed with status (VALID) at peer0-org2:7051
2020-04-16 10:23:01.839 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200
```

查询宝石信息：

```
peer chaincode query -C mychannel -n marbles -c '{"Args":["readMarble","marble2"]}'

# 显示如下
{"docType":"marble","name":"marble2","color":"red","size":50,"owner":"tom"}
```

也可以执行如下命令查询链码日志：

```
oc logs chaincode-marbles-org1-7496769f86-gqsw2

# 显示如下
invoke is running initMarble
- start init marble
- end init marble
invoke is running initMarble
- start init marble
- end init marble
invoke is running initMarble
- start init marble
This marble already exists: marble2
```

## 参考链接

https://medium.com/swlh/how-to-implement-hyperledger-fabric-external-chaincodes-within-a-kubernetes-cluster-fd01d7544523

http://blog.hubwiz.com/2020/03/12/fabric-2-external-chaincode/