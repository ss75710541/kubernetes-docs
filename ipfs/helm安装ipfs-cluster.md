# helm安装ipfs-cluster

## 添加repo

```sh
helm repo add paradeum-team https://paradeum-team.github.io/helm-charts/
```

## 下载chart

```sh
mkdir ~/ipfs-cluster
cd ~/ipfs-cluster
helm pull paradeum-team/ipfs-cluster --version 0.0.15
```

## 创建tls secret

```sh
kubectl create ns ipfs
kubectl create secret tls example-com-tls --cert=example.com.pem --key=example.com.key -n ipfs
```

## 创建values.yaml

```yaml
#### secretes #####
#https://cluster.ipfs.io/documentation/guides/k8s/

#od  -vN 32 -An -tx1 /dev/urandom | tr -d ' \n'
clusterSecret: 1ec8276f98cf47c16acfd9bf39fca38f8e3cfcbe229530a7ba9f08ef9757c439

#go get github.com/whyrusleeping/ipfs-key
# ipfs-key --type Ed25519 | base64 -w 0
bootstrapPeerPrivateKey: CAESQKyMCUbsfSRq8NBOFQOxv9uvgXm1zvSHyThj3AQV6UBHvTJ+TbTrk1Z6639aE6FOSMGbAG+besQOtk5SPsP2Gxo=
bootstrapPeerId: 12D3KooWNYut1XL31b4KUnCZmC8Mu7WqGn6QdwnptGpS5tnhSttR

clusterRestApiId: 12D3KooWMfXzp2nmNrb7DM4PETYZbaKALnrnwiqnhvrUC66KyYrb
clusterRestApiPrivateKey: "CAESQEmvGJbMboEibpcWCTKOtDYU2eEyyHLN9gDdJli6Z2tksAkhFWNx0Fk3vOlwLIitE2rfGtIj61Ovla/mHC42Plg="
clusterRestApiBasicAuth: "pld:password"

clusterCRDTtrustedPeers: "*"
#########################
clusterMonitorPingInterval: 2s

replicaCount: 3
# p2p svc loadblance外部IP
serviceExternalIPs:
  - x.x.x.x
  - 172.17.17.64

ingress:
  className: nginx
  # ipfs gateway ingress
  gateway:
    enabled: true
    host: dev-ipfs-gateway.example.com
    secretName: example-com-tls
  # ipfs cluster api ingress
  clusterApi:
    enabled: true
    host: dev-ipfs-cluster-api.example.com
    secretName: example-com-tls

persistence:
  enabled: true
  clusterStorage: 5Gi
  ipfsStorage: 20Gi

monitor:
  enabled: true
  app: kube-prometheus-stack  # prometheus operator app
  release: prometheus-community # prometheus operator release

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: "app"
            operator: In
            values:
            - "dev-ipfs-cluster"
        topologyKey: "kubernetes.io/hostname"
```

## 执行安装

```
helm upgrade --install dev-ipfs-cluster ipfs-cluster-0.0.15.tgz -f values.yaml -n ipfs
```