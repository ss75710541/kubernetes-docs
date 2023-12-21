# helm  安装openfaas

## 安装arkade

```sh
wget https://github.com/alexellis/arkade/releases/download/0.10.17/arkade
chmod +x arkade
mv arkade /usr/local/bin/arkade
```

## 安装faas-cli

```sh
arkade get faas-cli
mv ~/.arkade/bin/faas-cli /usr/local/bin/
# 自动补全设置到bashrc 中，如果使用了其它sh 需要修改
grep "faas-cli completion --shell bash" ~/.bashrc || echo "source <(faas-cli completion --shell bash)" >> ~/.bashrc
faas-cli version
```

## 创建namespace

```sh
wget https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
kubectl apply -f namespaces.yml
```

## 拉取openfaas chart

```sh
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm update
helm pull openfaas/openfaas
```

## 编写values.yaml

```yaml
# ingress configuration
ingress:
  enabled: true
  hosts:
    - host: gateway-openfaas.example.com
      serviceName: gateway
      servicePort: 8080
      path: /
  tls:
    - secretName: example-com-tls
      hosts:
        - gateway-openfaas.example.com
  ingressClassName: nginx
```

## 安装openfaas

```sh
helm upgrade openfaas \
  --install openfaas-14.2.1.tgz \
  --namespace openfaas
```

显示如下

```sh
Release "openfaas" does not exist. Installing it now.
NAME: openfaas
LAST DEPLOYED: Fri Dec  8 18:34:25 2023
NAMESPACE: openfaas
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To verify that openfaas has started, run:

  kubectl -n openfaas get deployments -l "release=openfaas, app=openfaas"

To retrieve the admin password, run:

  echo $(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
```

## 参考

https://artifacthub.io/packages/helm/openfaas/openfaas

https://github.com/openfaas/faas-netes/tree/master/chart/openfaas

https://docs.openfaas.com/deployment/kubernetes/

https://www.openfaas.com/blog/bring-gitops-to-your-openfaas-functions-with-argocd/