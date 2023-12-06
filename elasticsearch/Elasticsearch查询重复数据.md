# Elasticsearch查询重复数据

## 查询重复数据

```sh
GET /web-rss-data/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ]
    }
  },
  "aggs": {
    "unique_records": {
      "terms": {
        "field": "cid",
        "min_doc_count": 2,
        "size": 100000
      }
    }
  }
}
```

