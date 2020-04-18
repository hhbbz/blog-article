---
title: 关于ES字符串类型Text和keyword
date: 2018-10-19 16:30:15
categories: 
- 后端
tags:
- Elasticsearch
- Java
---

# 问题描述

今天在做es的聚合查询的时候，遇到了一个问题。

```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "megacorp",
        "node": "-Md3f007Q3G6HtdnkXoRiA",
        "reason": {
          "type": "illegal_argument_exception",
          "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
        }
      }
    ],
    "caused_by": {
      "type": "illegal_argument_exception",
      "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
    }
  },
  "status": 400
}
```

搜了一下应该是5.x后对排序，聚合这些操作用单独的数据结构(fielddata)缓存到内存里了，需要单独开启。

# 解决方法

遇到这个错误是因为你尝试对一个text类型的字段做排序，而text类型的字段是要分词的。 一来词典很大，性能会很差；二来排序结果是词典里的词，而并非整个text的内容。 出于这2点原因，ES5.x以后对于text类型默认禁用了fielddata，防止对text字段一些错误的操作（排序，聚合，script)而给heap造成很大的压力。如果一定有对该字段按照文本字母序做排序的需求，可以将该字段定义为multi-filed。于是我选择了将字段定义为keyword。keyword字段是通过doc values排序的，内存消耗远小于fielddata。

# Text和keyword的区别

ElasticSearch 5.0以后，string类型有重大变更，移除了string类型，string字段被拆分成两种新的数据类型: text用于全文搜索的,而keyword用于关键词搜索。
ElasticSearch字符串将默认被同时映射成text和keyword类型，将会自动创建下面的动态映射(dynamic mappings):

```json

{

    "foo": {

        "type": "text",

        "fields": {

            "keyword": {

                "type": "keyword",

                "ignore_above": 256

            }

        }

    }

}
```

这就是造成部分字段还会自动生成一个与之对应的“.keyword”字段的原因。

Text：

* 会分词，然后进行索引

* 支持模糊、精确查询

* 不支持聚合

keyword：

* 不进行分词，直接索引

* 支持模糊、精确查询

* 支持聚合

# 结论

keyword类型满足目前系统的需求，且能保证接口统一，所以建议将查询、聚合中涉及的字符串类型，在mapping中设置为“keyword”。