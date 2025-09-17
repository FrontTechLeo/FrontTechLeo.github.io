---
title: elasticsearch使用总结
name: Z.C Yang
date: 2025-09-16 10:00:00 -0500
categories: [后端]
tags: [elasticsearch]
---

## kibana控制台操作elasticsearch

- 查看所有节点：

```bash
GET _cat/node
```

- 查看所有索引

```bash
GET _cat/indices?v
```

- 创建book索引（无crateTime字段，匹配现有数据结构）

```bash
PUT book
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "page": {
        "type": "integer"
      },
      "content": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

- 查看book索引数据

```bash
GET book/_search
{
  "query": {
    "match": {
      "content": "jack"
    }
  }
}
```

- 添加一条数据

```bash
POST book/_doc
{
  "page": 8,
  "content": "jack喜欢运动"
}
```

- 更新数据

```bash
PUT book/_doc/yikqWJkBtsl_BYdV0VRk
{
  "page": 8,
  "content": "jack喜欢运动, 不只是跑步"
}

```

- 删除数据

```bash
POST book/_delete_by_query
{
  "query": {
    "match": {
      "page": 22
    }
  }
}
```


- 批量插入数据

```bash
POST book/_bulk
{ "index":{} }
{ "page":22 , "content": "Adversity, steeling will strengthen body.逆境磨练意志，锻炼增强体魄。"}
{ "index":{} }
{ "page":23 , "content": "Reading is to the mind.读书之于头脑，好比运动之于身体。"}
```
