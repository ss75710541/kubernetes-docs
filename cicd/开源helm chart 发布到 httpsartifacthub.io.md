# 开源helm chart 发布到 https://artifacthub.io/

## 注册登录https://artifacthub.io/

## 添加 repository

1. 登录后点头像菜单--> Control Panel 

2. 点击 ADD

3. 填写表单

   | 名称                | 值                                           |
   | ------------------- | -------------------------------------------- |
   | Kind                | Helm Charts                                  |
   | Name                | paradeum-team                                |
   | Url (helm 仓库地址) | https://paradeum-team.github.io/helm-charts/ |

   点击ADD

4. Helm-charts 仓库中根目录添加 **artifacthub-repo.yml**

   ```yaml
   repositoryID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx # 已经在 Artifact Hub 添加的 repository ID
   owners: # (optional, used to claim repository ownership)
     - name: user1
       email: user1@email.com
     - name: user2
       email: user2@email.com
   ignore: # (optional, packages that should not be indexed by Artifact Hub)
     - name: package1
     - name: package2 # Exact match
       version: beta # Regular expression (when omitted, all versions are ignored)
   ```

5. 本示例helm-charts仓库使用github-pages 提供web 服务 

   - 如果有自建其它仓库也可以

   - 同时 Artifact Hub 也支持 oci registry  接口仓库

6. 打包helm chart 并更新helm charts 仓库

   下面示例仅供参考

   ```sh
   mkdir ~/github
   cd ~/github
   git clone https://github.com/paradeum-team/geth-helm-charts.git
   cd geth-helm-charts
   helm package walletconnect-relay
   
   # 显示如下
   Successfully packaged chart and saved it to: /Users/liujinye/github/geth-helm-charts/walletconnect-relay-0.1.2.tgz
   ```

   移动chart 到 helm-charts git 仓库

   ```sh
   cd ~/github
   git clone https://github.com/paradeum-team/helm-charts.git
   mv ~/github/geth-helm-charts/walletconnect-relay-0.1.2.tgz ~/github/helm-charts/walletconnect-relay/
   cd ~/github/helm-charts/
   # 生成helm repo 索引
   helm repo index  . --url https://paradeum-team.github.io/helm-charts
   # 提交记录到线上
   git add .
   git commit -m "add walletconnect-relay-0.1.2.tgz"
   git push
   ```

7. Artifact Hub 会定时从 helm-charts 仓库更新最新的helm chart, 然后就可以在 Artifact Hub 看到发布的 chart 项目了 https://artifacthub.io/packages/helm/paradeum-team/walletconnect-relay

## 参考：

https://artifacthub.io/docs/topics/repositories/helm-charts/

https://artifacthub.io/docs/topics/annotations/helm/

https://github.com/artifacthub/hub/blob/master/docs/metadata/artifacthub-repo.yml

https://docs.github.com/cn/pages/getting-started-with-github-pages