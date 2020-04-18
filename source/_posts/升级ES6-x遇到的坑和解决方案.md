---
title: 升级ES6.x遇到的坑和解决方案
date: 2019-03-10 15:36:10
categories: 
- 后端
tags:
- Elasticsearch
- Java
---

# 问题描述

这星期公司要把es5.x升级到es6.x，因为是一个大版本的升级，所以坑肯定是不可避免的，详细的变更可以通过链接了解。[ES各版本变更信息](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/es-release-notes.html)。

这里我主要讲对我有较大影响的两个地方：

1. ES官方建议通过Java Low Level REST Client来访问Elasticsearch，不建议通过TransportClient来访问。

2. ES6.x起，同个_index不再支持多个_type.

3. ES6.x起，可以支持使用sql进行查询。

# 使用REST Client

ES官方已不建议通过TransportClient来访问Elasticsearch，使用TransportClient 5.5.3版在建立连接时会报 NoNodeAvailableException 问题，并且ES官方已经不再维护TransportClient。如果已经使用并且有提示报错，可用以下方法绕过：

* 建议换成[Java Low Level REST Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-low.html)来访问Elasticsearch。
* 在Java maven项目的POM配置文件中，使用 x-pack-transport-5.3.3 版client进行访问，并且POM配置文件中的elasticsearch版本也必须为5.3.3版，不建议通过该方案来绕着实现，不保证能完全兼

Java Low Level REST Client示例代码

* {USER NAME}：替换为访问ES实例对应用户名。
* {PASSWORD}：替换为访问ES实例指定用户对应密码。
* {HOST}:替换为ES实例基本信息界面中的内网或外网地址。
  
```java
public class RestClientTest {
    public  static void main(String[]args){
        final RequestOptions COMMON_OPTIONS;
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        //设置用户名密码
        credentialsProvider.setCredentials(AuthScope.ANY,
                new UsernamePasswordCredentials("{USER NAME}", "{PASSWORD}"));
        RestClient restClient = RestClient.builder(new HttpHost("{HOST}", 9200))
                .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                    @Override
                    public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                        return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                    }
                }).build();
        //如果需要构造Authorization，在下面request.setOptions();
        RequestOptions.Builder builder = RequestOptions.DEFAULT.toBuilder();
        builder.setHttpAsyncResponseConsumerFactory(
            new HttpAsyncResponseConsumerFactory
                .HeapBufferedResponseConsumerFactory(104857600));
        builder.addHeader("Authorization", "Bearer " + "123456");
        COMMON_OPTIONS = builder.build();
        try {
            //index a document
            HttpEntity entity = new NStringEntity("{\n\"user\" : \"hhbbz\"\n}", ContentType.APPLICATION_JSON);
            Response indexResponse = restClient.performRequest(
                    "PUT",
                    "/school/user/123",
                    Collections.<String, String>emptyMap(),
                    entity);
            //search a document
            Response response = restClient.performRequest("GET", "/school/user/123",
                    Collections.singletonMap("pretty", "true"));
            System.out.println(EntityUtils.toString(response.getEntity()));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

一般来说REST Client的相关api是不足以满足我们日常使用ES的操作的，所以我们需要使用RestHighLevelClient进行更多样化的操作。可以参考下面文档进行使用。

[RestHighLevelClient中文文档](https://my.oschina.net/GinkGo/blog/1853345#h1_1)

[RestHighLevelClient英文官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html)

# 一个_index不能有多个_type

## 原因分析

官方给出了一些说法，这里也总结下，无非就是以下几点：

1. 将 index , doc_type 类比为 RDBMS 的 db , table 这个出发点本身就是不对的。

2. 除了造成初学者的困惑，另外一个原因就是，在传统db中，不同的表中可以有相同命名但是不同数据类型、不同含义的字段，但是到了es中，这就麻烦了。

假设你有一个索引名叫做blog，其中 type 为 **user** 的 **deleted** 字段类型为boolean，又想在type为 **articles** 的 **deleted** 创建字段类型为date，那么你就麻烦了，ES是不允许这样做的。
究其原因是因为，在 Lucene 底层实现上，同一索引的不同type中的相同字段，存储结构都是相同的。

最重要的是，在同一索引中存储具有少量或不共有字段的不同实体，会导致数据稀疏，并干扰Lucene高效压缩文档的能力。

## 兼容方案

官方有给出兼容的方案，就是通过reindex的方式来将现有数据中的_type字段数据转移到type字段中存储。

```json
POST _reindex
{
  "source": {
    "index": "old"
  },
  "dest": {
    "index": "new"
  },
  "script": {
    "inline": """
    ctx._id = ctx._type + "-" + ctx._id;
    ctx._source.type = ctx._type;
    ctx._type = "doc";
"""
  }
}
```

这会将特殊的“_type”字段移动到“type”，然后您可以在后续过滤，聚合等中使用它。

## 后续开发解决方案

建议在文档中手动添加一个字段为 type，在查询的时候匹配即可，至于_type字段的内容，给个默认值就行。

# SQL查询的支持

在ES6.x中，我们可以使用SQL来进行查询数据，ES引擎内部会把SQL转换成DSL的。

DSL示例代码：

```json
POST /_xpack/sql?format=json
{
    "query": "SELECT * FROM library ORDER BY page_count DESC LIMIT 5"
}
```

返回值

```json
             {
                  "columns": [
                      {"name": "author",       "type": "text"},
                      {"name": "name",         "type": "text"},
                      {"name": "page_count",   "type": "short"},
                      {"name": "release_date", "type": "date"}
                  ],
                  "rows": [
                      ["Peter F. Hamilton",  "Pandora's Star",       768, "2004-03-02T00:00:00.000Z"],
                      ["Vernor Vinge",       "A Fire Upon the Deep", 613, "1992-06-01T00:00:00.000Z"],
                      ["Frank Herbert",      "Dune",                 604, "1965-06-01T00:00:00.000Z"],
                      ["Alastair Reynolds",  "Revelation Space",     585, "2000-03-15T00:00:00.000Z"],
                      ["James S.A. Corey",   "Leviathan Wakes",      561, "2011-06-02T00:00:00.000Z"]
                  ],
             }
```

REST Client示例代码：

```java
    String sql = "SELECT * FROM school";
    Request request = new Request("POST", "/_xpack/sql?format=json");
    HttpEntity entity = new NStringEntity("{\"query\": \"" + sql + "\"}", ContentType.APPLICATION_JSON);
    request.setEntity(entity);
    Response response = restClient.performRequest(request);
    System.out.println(EntityUtils.toString(response.getEntity()));
```

支持的SQL函数只有链接里面这一些，[ES支持的SQL函数](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/sql-syntax-show-functions.html)

具体的有关SQL的支持，可以通过查看官方文档来进行了解：
[ES对SQL的支持](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/xpack-sql.html)


