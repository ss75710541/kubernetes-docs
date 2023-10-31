# 使用kibana修改数据流索引mapping

## 修改索引模板mapping

```json
{
    "dynamic_templates": [],
    "properties": {
      "@timestamp": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_second"
      },
      ...
      "link": {
        "type": "keyword",
        "normalizer": "lowercase"
      }
    }
}
```

## 数据流滚动

打 Kibana dev tools ，手动触发滚动数据流索引

```sh
POST web-rss-data/_rollover
```

结果显示如下，创建新索引为`.ds-web-rss-data-2023.10.31-000004`

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "old_index": ".ds-web-rss-data-2023.10.31-000002",
  "new_index": ".ds-web-rss-data-2023.10.31-000004",
  "rolled_over": true,
  "dry_run": false,
  "conditions": {}
}
```

数据流滚动后，新创建的index 使用新的mapping设置

## 使用reindex把旧索引数据导入到数据流

因为是reindex 导入到数据流所以`op_type` 为 `create`

```json
POST _reindex
{
  "source": {
    "index": ".ds-web-rss-data-2023.10.31-000002"
  },
  "dest": {
    "index": "web-rss-data",
    "op_type": "create"
  }
}
```

返回类似这样数据

```json
{
  "took": 48,
  "timed_out": false,
  "total": 61,
  "updated": 0,
  "created": 61,
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1,
  "throttled_until_millis": 0,
  "failures": []
}
```

## 删除旧索引

**注意：删除前从Kibana 查看该索引是否为旧mapping, 如果该索引本身就是新mapping ，reindex 后索引名称和数据还都在原索引中，想瘦于reindex 后没有变化，这种情况就不能旧的索引**

### 删除旧索引

```sh
DELETE .ds-web-rss-data-2023.10.31-000002
```

返回类似这样数据

```
{
  "acknowledged": true
}
```

