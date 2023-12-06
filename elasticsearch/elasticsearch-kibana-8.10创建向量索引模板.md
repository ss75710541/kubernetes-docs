# elasticsearch-kibana-8.10创建向量索引模板

## 尝试kibana 管理页面创建索引模板 

kibana 索引模板页面 mapping 设置密集向量属性中不支持设置 `index 和 ` similarity 变量

## 使用kibana 开发工具创建或更新索引模板

```json
PUT /_index_template/web-rss-ai-data
{
  "index_patterns": ["web-rss-ai-data"],
  "data_stream": {},
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "web-rss-data",
          "rollover_alias": "web-rss-data"
        },
        "routing": {
          "allocation": {
            "include": {
              "_tier_preference": "data_hot"
            }
          }
        },
        "analysis": {
          "normalizer": {
            "lowercase": {
              "filter": [
                "lowercase"
              ],
              "type": "custom",
              "char_filter": []
            }
          }
        },
        "number_of_shards": "1",
        "number_of_replicas": "1"
      }
    },
    "mappings": {
      "dynamic_templates": [],
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "strict_date_optional_time||epoch_second"
        },
        "appId": {
          "type": "keyword"
        },
        "authors": {
          "type": "keyword"
        },
        "categories": {
          "type": "keyword"
        },
        "channelId": {
          "type": "integer"
        },
        "cid": {
          "type": "keyword"
        },
        "createdDate": {
          "type": "date"
        },
        "description": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "feedId": {
          "type": "keyword"
        },
        "language": {
          "type": "keyword",
          "normalizer": "lowercase"
        },
        "link": {
          "type": "keyword",
          "normalizer": "lowercase"
        },
        "model": {
          "type": "keyword",
          "normalizer": "lowercase"
        },
        "publishDate": {
          "type": "date"
        },
        "sourceId": {
          "type": "integer"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "titleVector": {
          "type": "dense_vector",
          "dims": 1536,
          "index": true,
          "similarity": "cosine"
        },
        "usagePromptTokens": {
          "type": "integer"
        },
        "usageTotalTokens": {
          "type": "integer"
        }
      }
    },
    "aliases": { }
  }
}
```



返回内容

```json
#! index template [web-rss-ai-data] has index patterns [web-rss-ai-data] matching patterns from existing older templates [web-rss-ai-data] with patterns (web-rss-ai-data => [web-rss-ai-data]); this template [web-rss-ai-data] will take precedence during new index creation
{
  "acknowledged": true
}
```

