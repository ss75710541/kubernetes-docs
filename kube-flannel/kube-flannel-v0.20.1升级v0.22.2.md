# kube-flannel-v0.20.1升级v0.22.2

下载flannel.yml

```curl
curl -o kube-flannel-v0.22.2.yaml  https://raw.githubusercontent.com/flannel-io/flannel/v0.22.2/Documentation/kube-flannel.yml
```

酌情修改配置，Network 的范围必须大于kubeadm-config 中 podSubnet 和 serviceSubnet 配置配置

```yaml
  ...
  net-conf.json: |
    {
      "Network": "10.0.0.0/8",
      "Backend": {
        "Type": "vxlan"
      }
    }
   ...
      # 下面配置酌情参考，不需要的可以忽略
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: rancher/mirrored-flannelcni-flannel:v0.20.1
        ...
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth0 # 按需配置，跨外网Node使用，不需要可以忽略
        - --public-ip=$(PUBLIC_IP) # 跨外网Node使用添加，不需要的可以忽略
        ...
        env:
        - name: PUBLIC_IP # 跨外网Node使用添加，不需要的可以忽略
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
```

更新 flannel v0.22.2

```sh
kubectl apply -f kube-flannel-v0.22.2.yaml
```

