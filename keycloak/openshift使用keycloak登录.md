# openshift使用keycloak登录

参考：

http://blog.keycloak.org/2015/06/openshift-ui-console-authentication.html

https://hub.docker.com/r/jboss/keycloak/

https://uptoknow.github.io/post/openshift-with-keycloak-openid/

http://blog.keycloak.org/2018/05/keycloak-on-openshift.html



https://docs.okd.io/3.11/install_config/configuring_authentication.html#OpenID

## 创建project

```
oc new-project keycloak
```

## 发布mysql

应用目录选择发布持久化Mysql 5.7

注意：创建时填写默认数据库名称为`keycloak`

## 发布keycloak

```
oc process -n keycloak  -f https://raw.githubusercontent.com/ss75710541/keycloak-with-openshift-auth-provider/master/keycloak-with-openshift-auth-provider.yaml -p KEYCLOAK_IMAGE="docker.io/jboss/keycloak:4.8.3.Final" | oc create -f -
```

修改keycloak环境变量, 增加下面环境变量

```
            - name: DB_DATABASE
              valueFrom:
                secretKeyRef:
                  key: database-name
                  name: mysql
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  key: database-user
                  name: mysql
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: database-password
                  name: mysql
            - name: DB_ADDR
              value: mysql.keycloak.svc
            - name: DB_PORT
              value: '3306'
            - name: DB_VENDOR
              value: mysql
            - name: MYSQL_PORT
              value: '3306'
```


## 修改keycloak证书

生成keycloak相关证书(一般使用tls证书)

### tls证书（默认选择，keycloak 镜像自动转为java使用的证书）

```
#!/bin/bash

domain="keycloak-https-keycloak.apps181.hisun.com"

project=keycloak

# 生成ca key

openssl genrsa -out $project-ca.key 2048

# 创建根证书

openssl req -utf8 -new -nodes -x509 -days 3650 -key $project-ca.key  -out $project-ca.crt -subj "/C=CN/ST=北京/L=北京/O=高阳金信/OU=IT/CN=$domain"

# 创建服务key

openssl genrsa -out $project.key 2048

# 创建服务证书

openssl req -utf8 -new -key $project.key -out $project.csr -subj "/C=CN/ST=北京/L=北京/O=高阳金信/OU=IT/CN=$domain"

# 签名证书

openssl x509 -req -in $project.csr -CA $project-ca.crt -CAkey $project-ca.key -CAcreateserial -out $project.crt -days 3650

# 生成pem

cat $project.crt $project.key > $project.pem
```

创建configmap（文件名称必须为tls.crt 和 tls.key ,否则不识别,所以在创建configmap前先修改文件名称）

```
oc create configmap keycloak-certs  --from-file=tls.crt --from-file=tls.key  -n keycloak
```

添加configmap挂载配置

```
...
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /etc/x509/https
              name: keycloak-certs
...
              
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 420
            name: keycloak-certs
          name: keycloak-certs
...
```

### 创建jks证书（安装使用麻烦，只做参考，弃用）

```
domain=keycloak-https-keycloak.apps181.hisun.com
passwd=password
project=keycloak

# 生成ca key

openssl genrsa -out $project-ca.key 2048

# 创建根证书

openssl req -utf8 -new -nodes -x509 -days 3650 -key $project-ca.key  -out $project-ca.crt -subj "/C=CN/ST=Beijing/L=Beijing/O=Hisun/OU=IT/CN=$domain"


keytool -genkey -alias server -keyalg RSA -keystore keycloak.jks -validity 10950 -keypass $passwd -storepass $passwd -dname "CN=$domain, OU=IT, O=Hisun, L=Beijing, ST=Beijing, C=CN"

keytool -storepass $passwd -certreq -alias server -keystore keycloak.jks  > keycloak.careq
cat keycloak.careq

openssl x509 -req -in keycloak.careq -CA $project-ca.crt -CAkey $project-ca.key -CAcreateserial -out $project.crt -days 500

keytool -import -keystore keycloak.jks -file $project-ca.crt -alias root -keypass $passwd -storepass $passwd

keytool -import -alias server -keystore keycloak.jks -file $project.crt -keypass $passwd -storepass $passwd
```

创建configmap

```
oc create configmap keycloak  --from-file=keycloak.jks --from-file=standalone-ha.xml -n keycloak
```

添加configmap挂载配置

```
...
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /opt/jboss/keycloak/standalone/configuration/application.keystore
              name: keycloak
              subPath: keycloak.jks
...
              
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 420
            name: keycloak
          name: keycloak
...
```

## 导入openshift realm

访问`https://keycloak-https-keycloak.apps181.hisun.com `

以admin账号登录keycloak ，导入[realm-openshift.json](https://raw.githubusercontent.com/ss75710541/openshift-docs/master/keycloak/realm-openshift.json)

## 修改clients

### 修改Valid Redirect URIs

选择realm--> Openshift --> Clients --> Settings

修改`Valid Redirect URIs` (根据实际openshift的访问地址修改)

```
https://master181.hisun.com:8443/*
```

### 重置keycloak openshift clinet 密钥

选择realm--> Openshift --> Clients --> Credentials 

点击 Regenerate secret 重置密钥

### 添加测试用户

选择realm--> Openshift --> Users --> Add user

### 重置新用户密码

选择 Users --> test --> Credentials

## 配置Openshift master

### 配置master config

登录master主机

cd /etc/origin/master/

上传keycloak-ca.crt文件

vi master-config.yaml

找到oauthConfig, identityProviders下添加下面内容

```
  identityProviders:
  - name: keycloak
    challenge: true
    login: true
    provider:
      apiVersion: v1
      kind: OpenIDIdentityProvider
      ca: keycloak-ca.crt
      clientID: openshift
      clientSecret: <填写上一步重置后的openshift client密钥>
      claims:
        id:
        - sub
        preferredUsername:
        - preferred_username
        name:
        - name
        email:
        - email
      urls:
        authorize: https://keycloak-https-keycloak.apps181.hisun.com/auth/realms/openshift/protocol/openid-connect/auth
        token: https://keycloak-https-keycloak.apps181.hisun.com/auth/realms/openshift/protocol/openid-connect/token
```

重启master api

```
master-restart api api
```

## 修改openshift logout url

编辑configmap webconsole-config

修改logoutPublicURL 的值为(酌情修改)

```
https://keycloak-https-keycloak.apps181.hisun.com/auth/realms/openshift/protocol/openid-connect/logout?redirect_uri=https://master181.hisun.com:8443/console
```
