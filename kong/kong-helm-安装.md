# kong helm 安装

## 创建kong gateway secret

1. 创建namespace:

   ```sh
   kubectl create namespace kong
   ```

   

2. 创建 Kong config 和 credential variables:

   ```sh
   kubectl create secret generic kong-config-secret -n kong \
       --from-literal=portal_session_conf='{"storage":"kong","secret":"super_secret_salt_string","cookie_name":"portal_session","cookie_same_site":"Lax","cookie_secure":false}' \
       --from-literal=admin_gui_session_conf='{"storage":"kong","secret":"super_secret_salt_string","cookie_name":"admin_session","cookie_same_site":"Lax","cookie_secure":false}' \
       --from-literal=pg_host="enterprise-postgresql.kong.svc.cluster.local" \
       --from-literal=kong_admin_password=kong \
       --from-literal=password=kong
   ```

   

3. 创建一 个Kong 企业免费版  license secret:

   ```sh
   kubectl create secret generic kong-enterprise-license --from-literal=license="'{}'" -n kong --dry-run=client -o yaml | kubectl apply -f -
   ```

## 安装 Cert Manager

1. 添加  Jetstack Cert Manager Helm 源:

   ```sh
   helm repo add jetstack https://charts.jetstack.io ; helm repo update
   ```

   

2. 安装 Cert Manager:

   在安装chart之前，必须先安装cert-manager CustomResourceDefinition资源。这是在一个单独的步骤中执行的，允许您轻松卸载和重新安装cert-manager，而不需要删除已安装的自定义资源。

   ```sh
   wget https://github.com/jetstack/cert-manager/releases/download/v1.11.2/cert-manager.crds.yaml -o cert-manager-v1.11.2.crds.yaml
   ```

   安装

   ```sh
   helm pull jetstack/cert-manager
   helm upgrade --install cert-manager cert-manager-v1.11.2.tgz \
       --set installCRDs=false --namespace cert-manager --create-namespace
   ```

   

3. 创建自签名证书 issuer:

   ```yaml
   bash -c "cat <<EOF | kubectl apply -n kong -f -
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: test-kong-selfsigned-issuer-root
   spec:
     selfSigned: {}
   ---
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: test-kong-selfsigned-issuer-ca
   spec:
     commonName: test-kong-selfsigned-issuer-ca
     duration: 2160h0m0s
     isCA: true
     issuerRef:
       group: cert-manager.io
       kind: Issuer
       name: test-kong-selfsigned-issuer-root
     privateKey:
       algorithm: ECDSA
       size: 256
     renewBefore: 360h0m0s
     secretName: test-kong-selfsigned-issuer-ca
   ---
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: test-kong-selfsigned-issuer
   spec:
     ca:
       secretName: test-kong-selfsigned-issuer-ca
   EOF"
   ```

## 部署 Kong Gaeway

1. 添加 Kong Helm repo:

   ```sh
   helm repo add kong https://charts.konghq.com ; helm repo update
   ```

   

2. Install Kong:

   创建values.yaml

   ```yaml
   admin:
     annotations:
       konghq.com/protocol: https
     enabled: true
     http:
       enabled: false
     ingress:
       annotations:
         konghq.com/https-redirect-status-code: "301"
         konghq.com/protocols: https
         konghq.com/strip-path: "true"
         nginx.ingress.kubernetes.io/app-root: /
         nginx.ingress.kubernetes.io/backend-protocol: HTTPS
         nginx.ingress.kubernetes.io/permanent-redirect-code: "301"
       enabled: true
       ingressClassName: kong
       hostname: kong.example.com
       path: /api
       tls: example-com-tls
     tls:
       containerPort: 8444
       enabled: true
       parameters:
       - http2
       servicePort: 8444
     type: ClusterIP
   affinity:
     podAntiAffinity:
       preferredDuringSchedulingIgnoredDuringExecution:
       - podAffinityTerm:
           labelSelector:
             matchExpressions:
             - key: app.kubernetes.io/instance
               operator: In
               values:
               - dataplane
           topologyKey: kubernetes.io/hostname
         weight: 100
   certificates:
     enabled: true
     issuer: test-kong-selfsigned-issuer
     cluster:
       enabled: true
     admin:
       enabled: true
       commonName: kong.example.com
     portal:
       enabled: false
       commonName: developer.example.com
     proxy:
       enabled: true
       commonName: example.com
       dnsNames:
       - '*.example.com'
   cluster:
     enabled: true
     labels:
       konghq.com/service: cluster
     tls:
       containerPort: 8005
       enabled: true
       servicePort: 8005
     type: ClusterIP
   clustertelemetry:
     enabled: true
     tls:
       containerPort: 8006
       enabled: true
       servicePort: 8006
       type: ClusterIP
   deployment:
     kong:
       daemonset: false
       enabled: true
   serviceMonitor:
     enabled: true
     interval: 15s
     labels:
       release: prometheus-community
       
   # securityContext for containers. 如果配置了自定义插件，要把容器设置为非只读权限，否则插件的socket文件不能创建
   containerSecurityContext:
     readOnlyRootFilesystem: false
   enterprise:
     enabled: true
     license_secret: kong-enterprise-license
     portal:
       enabled: false
     rbac:
       admin_api_auth: basic-auth
       admin_gui_auth_conf_secret: kong-config-secret
       enabled: true
       session_conf_secret: kong-config-secret
     smtp:
       enabled: false
     vitals:
       enabled: false
   env:
     admin_access_log: /dev/stdout
     admin_api_uri: https://kong.example.com/api
     admin_error_log: /dev/stdout
     admin_gui_access_log: /dev/stdout
     admin_gui_error_log: /dev/stdout
     admin_gui_host: kong.example.com
     admin_gui_protocol: https
     admin_gui_url: https://kong.example.com/
     cluster_data_plane_purge_delay: 60
     cluster_listen: 0.0.0.0:8005
     cluster_telemetry_listen: 0.0.0.0:8006
     database: "postgres"
     log_level: debug
     lua_package_path: /opt/?.lua;;
     nginx_worker_processes: "2"
     password:
       valueFrom:
         secretKeyRef:
           key: kong_admin_password
           name: kong-config-secret
     pg_database: kong
     pg_host:
       valueFrom:
         secretKeyRef:
           key: pg_host
           name: kong-config-secret
     pg_ssl: "off"
     pg_ssl_verify: "off"
     pg_user: kong
     plugins: bundled,openid-connect
     # 如果有自定义插件下面内容替换plugins 配置，同时要注意containerSecurityContext配置
     #plugins: "bundled,openid-connect,plugin-custom"
     #pluginserver_names: "plugin-custom"
     #pluginserver_plugin_bucket_start_cmd: "/usr/local/bin/plugin-custom"
     #pluginserver_plugin_bucket_query_cmd: "/usr/local/bin/plugin-custom -dump"
     portal: false
     #portal_api_access_log: /dev/stdout
     #portal_api_error_log: /dev/stdout
     #portal_api_url: https://developer.example.com/api
     #portal_auth: basic-auth
     #portal_cors_origins: '*'
     #portal_gui_access_log: /dev/stdout
     #portal_gui_error_log: /dev/stdout
     #portal_gui_host: developer.example.com
     #portal_gui_protocol: https
     #portal_gui_url: https://developer.example.com/
     #portal_session_conf:
     #  valueFrom:
     #    secretKeyRef:
     #      key: portal_session_conf
     #      name: kong-config-secret
     prefix: /kong_prefix/
     proxy_access_log: /dev/stdout
     proxy_error_log: /dev/stdout
     proxy_stream_access_log: /dev/stdout
     proxy_stream_error_log: /dev/stdout
     smtp_mock: "on"
     status_listen: 0.0.0.0:8100
     trusted_ips: 0.0.0.0/0,::/0
     vitals: "off"
     dns_order: "LAST,SRV,A,CNAME"
   
   extraLabels:
     konghq.com/component: test
   image:
     repository: kong/kong-gateway
     tag: "3.2"
   ingressController:
     enabled: true
     env:
       kong_admin_filter_tag: ingress_controller_kong
       kong_admin_tls_skip_verify: true
       kong_admin_token:
         valueFrom:
           secretKeyRef:
             key: password
             name: kong-config-secret
       kong_admin_url: https://localhost:8444
       kong_workspace: default
       publish_service: kong/test-kong-proxy
     image:
       repository: docker.io/kong/kubernetes-ingress-controller
       tag: "2.9"
     ingressClass: kong
     installCRDs: false
   manager:
     annotations:
       konghq.com/protocol: https
     enabled: true
     http:
       containerPort: 8002
       enabled: false
       servicePort: 8002
     ingress:
       annotations:
         konghq.com/https-redirect-status-code: "301"
         nginx.ingress.kubernetes.io/backend-protocol: HTTPS
       enabled: true
       ingressClassName: kong
       hostname: kong.example.com
       path: /
       tls: test-kong-admin-cert
     tls:
       containerPort: 8445
       enabled: true
       parameters:
       - http2
       servicePort: 8445
     type: ClusterIP
   migrations:
     enabled: true
     postUpgrade: true
     preUpgrade: true
   namespace: kong
   podAnnotations:
     kuma.io/gateway: enabled
   portal:
     annotations:
       konghq.com/protocol: https
     enabled: false
     http:
       containerPort: 8003
       enabled: false
       servicePort: 8003
     ingress:
       annotations:
         konghq.com/https-redirect-status-code: "301"
         konghq.com/protocols: https
         konghq.com/strip-path: "false"
       enabled: false
       ingressClassName: kong
       hostname: developer.example.com
       path: /
       tls: test-kong-portal-cert
     tls:
       containerPort: 8446
       enabled: true
       parameters:
       - http2
       servicePort: 8446
     type: ClusterIP
   portalapi:
     annotations:
       konghq.com/protocol: https
     enabled: false
     http:
       enabled: false
     ingress:
       annotations:
         konghq.com/https-redirect-status-code: "301"
         konghq.com/protocols: https
         konghq.com/strip-path: "true"
         nginx.ingress.kubernetes.io/app-root: /
       enabled: true
       ingressClassName: kong
       hostname: developer.example.com
       path: /api
       tls: test-kong-portal-cert
     tls:
       containerPort: 8447
       enabled: true
       parameters:
       - http2
       servicePort: 8447
     type: ClusterIP
   postgresql:
     enabled: true
     auth:
       database: kong
       username: kong
   proxy:
     annotations:
       prometheus.io/port: "9542"
       prometheus.io/scrape: "true"
     enabled: true
     http:
       containerPort: 8080
       enabled: true
       hostPort: 80
     ingress:
       enabled: false
     labels:
       enable-metrics: true
     tls:
       containerPort: 8443
       enabled: true
       hostPort: 443
     externalIPs:
       - x.x.x.x
     externalTrafficPolicy: Local # 配置这个是为了获取proxy 转发到后端的 remote_addr 为 真实client ip
     type: NodePort
   replicaCount: 1
   secretVolumes: []
   status:
     enabled: true
     http:
       containerPort: 8100
       enabled: true
     tls:
       containerPort: 8543
       enabled: false
   updateStrategy:
     rollingUpdate:
       maxSurge: 50%
       maxUnavailable: 50%
     type: RollingUpdate
   ```

   安装

   ```sh
   helm pull kong/kong
   helm upgrade --install test kong-2.20.1.tgz --namespace kong -f values.yaml 
   ```

   

3. 等待所有pod都处于“Running”和“Completed”状态:

   ```sh
   kubectl get po --namespace kong -w
   ```

   

4. 一旦所有pod都开始运行，在浏览器的入口主机域中打开Kong Manager，例如:[https://kong.example.com](https://kong.example.com/)。或者用下面的命令打开它:

   ```sh
   open "https://$(kubectl get ingress --namespace kong test-kong-manager -o jsonpath='{.spec.tls[0].hosts[0]}')"
   ```

   

   > 由于使用自签名证书，您将收到“您的连接不是私有的”警告消息。如果您使用的是Chrome浏览器，可能没有“接受风险并继续”选项，请在标签集中继续时键入“thisisunsafe”。

5. 免费版本默认没有认证直接访问即可，如果需要配置认证需要安装配置kong 认证插件

## 参考

https://docs.konghq.com/gateway/latest/install/kubernetes/helm-quickstart/

https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/preserve-client-ip/

https://docs.konghq.com/gateway/latest/plugin-development/pluginserver/go/

https://docs.konghq.com/gateway/latest/plugin-development/pluginserver/plugins-kubernetes/