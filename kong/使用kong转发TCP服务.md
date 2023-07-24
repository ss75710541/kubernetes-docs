#  使用kong转发TCP服务

## 修改helm values.yaml

```yaml
env:
  ...
  proxy_stream_access_log: "/dev/stdout basic" # 默认为 /dev/stdout，但是默认值报log_format 没有设置的错误，所以改为现在的值，basic 为kong 默认的log_format
  proxy_stream_error_log: "/dev/stdout"
  ...
...
proxy:
  ...
  stream:
    - containerPort: 9000
      servicePort: 9000
      protocol: "TCP"
    - containerPort: 9443
      servicePort: 9443
      protocol: "TCP"
      parameters:
      - ssl
...
```

更新kong 服务

```sh
helm upgrade --install test kong-2.20.2.tgz --namespace kong -f values.yaml
```

查看kong svc 转发端口

```
...
test-kong-proxy                NodePort    10.106.216.200   x.x.x.x,172.17.81.229   80:31360/TCP,443:31346/TCP,9000:32601/TCP,9443:31468/TCP   70d
...
```

## 安装 TCP echo 服务

```sh
kubectl apply -f https://docs.konghq.com/assets/kubernetes-ingress-controller/examples/tcp-echo-service.yaml
```

## 配置 echo tcp 转发

```yaml
apiVersion: configuration.konghq.com/v1beta1
kind: TCPIngress
metadata:
  name: echo-plaintext
  # namespace: test-echo, 默认当前，如果echo 服务不是在当前namespace ，设置目标ns
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - port: 9000
    backend:
      serviceName: tcp-echo
      servicePort: 2701
```

查看tcpingress

```sh
kubectl get  tcpingress
NAME             ADDRESS          AGE
echo-plaintext   10.106.216.200   9s
```

### 测试连接

```sh
# telnet 10.106.216.200 9000 # 或使用负载均衡 IP 连接
Trying 10.106.216.200  ...
Connected to 10.106.216.200.
Escape character is '^]'.
Welcome, you are connected to node node4.pldtest.k8s.
Running on Pod tcp-echo-58ccd6b78d-jtttt.
In namespace test-echo.
With IP address 10.128.6.49.
```

## 配置 域名+SSL TCP 转发

```yaml
apiVersion: configuration.konghq.com/v1beta1
kind: TCPIngress
metadata:
  name: echo-plaintext-ssl
  # namespace: test-echo, 默认当前，如果echo 服务不是在当前namespace ，设置目标ns
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - port: 9443
    host: echo-ssl.example.com
    backend:
      serviceName: tcp-echo
      servicePort: 2701
  tls:
  - hosts:
    - echo-ssl.example.com
    secretName: netwarps-com-tls
```

查看

````sh
kubectl get tcpingress

NAME                 ADDRESS          AGE
echo-plaintext       10.106.216.200   11m
echo-plaintext-ssl   10.106.216.200   6m57s
````



### 测试连接

```sh
PROXY_IP=10.106.216.200 # 或输入proxy 负载均衡IP
echo "hello" | openssl s_client -connect $PROXY_IP:9443 -servername echo-ssl.example.com -quiet 2>/dev/null
```

显示如下

```
Welcome, you are connected to node node4.pldtest.k8s.
Running on Pod tcp-echo-58ccd6b78d-jtttt.
In namespace test-echo.
With IP address 10.128.6.49.
hello
```



## 参考

https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/using-tcpingress/

https://stackoverflow.com/questions/75304912/how-to-expose-mysql-database-in-kubernetes-using-kong-gateway

