# rancher-backup使用US3备份

## 先决条件

Rancher 必须是 2.5.0 或更高版本。

请参见[此处](https://docs.ranchermanager.rancher.io/zh/how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/migrate-rancher-to-new-cluster#2-使用-restore-自定义资源来还原备份)获取在 Rancher 2.6.3 中将现有备份文件恢复到 v1.22 集群的帮助。

## 安装rancher-backup operator

### 创建secret

```
apiVersion: v1
data:
  accessKey: xxxxxxxxxxxxxxxxxxxxxxxxxxx
  secretKey: xxxxxxxxxxxxxxxxxxxxxxxxxxx
kind: Secret
metadata:
  managedFields:
  name: rancher-backup-s3-key
  namespace: cattle-system
type: Opaque

```

### 安装rancher-backup operator

1. 在左上角，单击 **☰ > 集群管理**。

2. 在**集群**页面上，转到 `local` 集群并单击 **Explore**。Rancher Server 运行在 `local` 集群中。

3. 单击**应用市场 > Chart**。

4. 点击 **Rancher 备份**。

5. 单击**安装**。

6. 配置默认存储位置。如需获取帮助，请参见[存储配置](https://docs.ranchermanager.rancher.io/zh/reference-guides/backup-restore-configuration/storage-configuration)。

   - 选择使用Amazon S3 对象存储服务
   - 密钥凭证选择 rancher-backup-s3-key
   - 桶名称 rancher-backup
   - 区域 s3-hk
   - 端点 s3-hk.ufileos.com 

     注意：如果使用AWS S3 SDK 或 s3minio SDK（rancher backup 使用） ，区域和端点 参考s3兼容endpoint  https://docs.ucloud.cn/ufile/s3/s3_introduction)

7. 单击**安装**。

##### NOTE

使用 `backup-restore` operator 执行恢复后，Fleet 中会出现一个已知问题：用于 `clientSecretName` 和 `helmSecretName` 的密文不包含在 Fleet 的 Git 仓库中。请参见[此处](https://docs.ranchermanager.rancher.io/zh/how-to-guides/new-user-guides/deploy-apps-across-clusters/fleet#故障排除)获得解决方法。

## 执行备份

要执行备份，必须创建 Backup 类型的自定义资源。

1. 在左上角，单击 **☰ > 集群管理**。
2. 在**集群**页面上，转到 `local` 集群并单击 **Explore**。
3. 在左侧导航栏中，点击 **Rancher 备份 > 备份**。
4. 单击**创建**。
5. 使用表单或 YAML 编辑器创建 Backup。
6. 要使用该表单配置 Backup 详细信息，请单击**创建**，然后参见[配置参考](https://docs.ranchermanager.rancher.io/zh/reference-guides/backup-restore-configuration/backup-configuration)和[示例](https://docs.ranchermanager.rancher.io/zh/reference-guides/backup-restore-configuration/examples#备份)进行操作。
7. 要使用 YAML 编辑器，单击**创建 > 使用 YAML 文件创建**。输入 Backup YAML。这个示例 Backup 自定义资源将在 S3 中创建不加密的定期备份。

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: backup-timing
spec:
  resourceSetName: rancher-resource-set # 必须rancher-resource-set，不能修改
  retentionCount: 360 # 保留个数
  schedule: 0 * * * * # cron 定时策略
```

8. 点击创建

   **结果**：备份文件创建在 Backup 自定义资源中配置的存储位置中。执行还原时使用该文件的名称。

## 参考：

https://docs.ranchermanager.rancher.io/zh/how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/back-up-rancher

https://docs.ranchermanager.rancher.io/zh/reference-guides/backup-restore-configuration/storage-configuration

https://docs.ucloud.cn/ufile/s3/s3_introduction

https://xie.infoq.cn/article/9c4e617568387037778ecefa3