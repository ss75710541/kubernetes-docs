# 阿里云 Mongodb副本集实例使用

## 版本 5.0 小版本 6.0.0-20210924172808_1

该版本不支持增加只读节点，只有三个节点（主、从、隐藏节点）

## 所有节点连接

```sh
mongodb://root:****@dds-8vb693ce0ee72dc41.mongodb.zhangbei.rds.aliyuncs.com:3717,dds-8vb693ce0ee72dc42.mongodb.zhangbei.rds.aliyuncs.com:3717/admin?replicaSet=mgset-509023933
```

**readPreference** 默认为 **primary** , 如果使用命令行连接 需要 调用 db.getMongo().setReadPref() 才能正常查询 从节点，示例如下

```sh
mongosh dds-8vbc36a3b73fcc442.mongodb.zhangbei.rds.aliyuncs.com:3717/admin -u root -p

> db.getMongo().setReadPref("secondary")
```

如果程序读取操作想要利用上从节点 的资源需要在 `ConnectionStringURI` 中 增加 `readPreference` 设置为 `secondary` 或 `secondaryPreferred`

## 版本4.2 小版本 mongodb_20210824_4.0.19

### 只读节点连接(只使用只读从节点)

```sh
mongodb://root:****@dds-8vb6215bc4a3e3443.mongodb.zhangbei.rds.aliyuncs.com:3717,dds-8vb6215bc4a3e3444.mongodb.zhangbei.rds.aliyuncs.com:3717/admin?readPreference=secondary&readPreferenceTags=role:readonly&replicaSet=mgset-509020673
```

### 所有节点连接

**readPreference** 默认为 **primary** (只使用主节点，如果需要利用从节点资源需要设置为其它模式)

```sh
mongodb://root:****@dds-8vb6215bc4a3e3441.mongodb.zhangbei.rds.aliyuncs.com:3717,dds-8vb6215bc4a3e3442.mongodb.zhangbei.rds.aliyuncs.com:3717,dds-8vb6215bc4a3e3443.mongodb.zhangbei.rds.aliyuncs.com:3717,dds-8vb6215bc4a3e3444.mongodb.zhangbei.rds.aliyuncs.com:3717/admin?replicaSet=mgset-509020673
```

## Read Preference 模式

- **primary** 主节点
  - 所有读取操作仅使用当前副本集主副本。 这是默认读取模式。如果主节点不可用，读取操作会产生错误或抛出异常。

- **primaryPreferred** 首选主节点
  - 在大多数情况下，操作从集合的主节点读取数据。但是，如果主节点不可用(就像故障转移期间的情况一样)，则从满足读首选项的 `maxStalenessSeconds` 和 tag 集的辅助节点读取操作。

- **secondary** 从节点
  - 操作只能从集合的从节点中读取。如果没有可用的从节点，则此读取操作会产生错误或异常。
- **secondaryPreferred** 首选从节点
  - 在大多数情况下，操作读取从节点数据，但在该集合由单个主节点（并且没有其他节点）组成的情况下，读取操作将使用副本集的主节点。
- **nearest** 最近节点
  - 驱动程序从网络延迟处于可接受延迟窗口内的节点读取数据。在路由读取操作时，最近模式的读取不考虑成员是主节点还是从节点:主节点和从节点被同等对待。

## 参考

https://www.mongodb.com/docs/manual/core/read-preference/

https://www.mongodb.com/docs/manual/reference/method/Mongo.setReadPref/