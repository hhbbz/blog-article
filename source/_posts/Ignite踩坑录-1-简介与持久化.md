---
title: Ignite踩坑录(1) 简介与持久化
date: 2019-05-19 14:07:35
categories: 
- 后端
tags:
- Ignite
- Java
- 分布式
---

# Ignite是什么

1. 一个以内存为中心的分布式数据库、缓存和处理平台，可以在PB级数据中，以内存级的速度进行事务性、分析性以及流式负载的处理。
2. 支持磁盘、第三方存储持久化数据。
3. 在内存和磁盘上是同时支持ACID的，是一个强一致的系统，Ignite可以在整个拓扑的多台服务器上保持事务。
4. 完整的SQL和键值支持。
5. 数据的并置计算。
6. 是一个弹性的、可水平扩展的分布式系统，它支持按需地添加和删除节点，Ignite还可以存储数据的多个副本，这样可以使集群从部分故障中恢复。

# ignite服务端和客户端
Ignite有一个可选的概念，就是客户端节点和服务端节点，服务端节点参与缓存、计算执行、流式处理等等，而原生的客户端节点提供了远程连接服务端的能力。Ignite原生客户端可以使用完整的Ignite API集合，包括近缓存、事务、计算、流、服务等等。
所有的Ignite节点默认都是以服务端模式启动的，客户端模式需要显式地启用。
可以通过IgniteConfiguration.setClientMode(...)属性配置一个节点，或者为客户端，或者为服务端。
```java
IgniteConfiguration cfg = new IgniteConfiguration();
// Enable client mode.
cfg.setClientMode(true);
// Start Ignite in client mode.
Ignite ignite = Ignition.start(cfg);
```

Ignite的唯一特点是所有节点都是平等的。没有master节点或者server节点，也没有worker节点或者client节点，按照Ignite的观点所有节点都是平等的。但是，可以将节点配置成主节点，工作节点，或者客户端以及数据节点



# 用Java来实现第一个Ignite应用

如果不需要对IgniteConfiguration进行设置的话，我们直接以默认的参数进行启动就行，也就是一行代码：

```java
Ignite ignite = Ignition.start();
```

这样就会以服务端节点的形式去启动一个ignite节点，然后加入到集群中。

>第一坑：如果以客户端/瘦客户端形式去启动，会无法根据业务去动态构建缓存，并且会因为ignite服务端节点中没有我们应用的代码，然后因为业务的配置出现classNotFoundException，所以这里我采用在应用中启动ignite服务端节点的做法。

然后我们需要对这个ignite节点进行构建缓存,来进行缓存操作。

Person.class

```java
@Getter
@Setter
public class Person{
    private Integer id;
    private String name;
}
```

```java
        //构造cacheConfig,可根据业务需要去动态进行构建
        CacheConfiguration ccf = new CacheConfiguration();
        //注意这里最好设置一个统一的schema
        ccf.setSqlSchema("public");
        ccf.setName("cacheName");
        ccf.setCacheMode(PARTITIONED);
        ccf.setAtomicityMode(CacheAtomicityMode.ATOMIC);
        ccf.setBackups(1);
        ignite.getOrCreateCache(ccf)
```

> 第二坑：如果不开启第三方持久化，并且没有配置QueryEntity的话，ignite不会进行自动建sql表，从而也就支持不了sql查询。
> 如果开启持久化的话，最好给一个默认的schema去让ignite建表，如图：

{% asset_img 002.png 图片 %}

> 第三坑：缓存配置一旦在集群中构建，便无法对其进行更新，如果确实有更新缓存配置的需要，只能通过destoryCache(),再createCache()的方法实现。

下面列举构建一个开启第三方持久化的缓存配置例子：

```java

        //构造cacheConfig
        CacheConfiguration ccf = new CacheConfiguration();
        ccf.setSqlSchema("public");
        ccf.setName("person");
        ccf.setCacheMode(PARTITIONED);
        ccf.setAtomicityMode(CacheAtomicityMode.ATOMIC);
       ccf.setWriteBehindEnabled(true);
       // 持久化后写每批次写入数量
       ccf.setWriteBehindBatchSize(10);
       //触发持久化后写的新增数据数量
       ccf.setWriteBehindFlushSize(10);
       //持久化后写刷新频率，即每过多少毫秒就持久化一次
       ccf.setWriteBehindFlushFrequency(10000);
       //持久化后写使用线程数量
       ccf.setWriteBehindFlushThreadCount(10);
       ccf.setReadThrough(true);
       ccf.setWriteThrough(true);
       //构造持久化类
       CacheJdbcPojoStoreFactory storeFactory = new CacheJdbcPojoStoreFactory();
       storeFactory.setDataSource(this.dataSource);
       storeFactory.setDialect(new BasicJdbcDialect());

       LinkedHashMap<String,String> fieldMap = new LinkedHashMap<>();

        //构造字段列表
       JdbcTypeField[] jdbcTypeFields = new JdbcTypeField[10];

       for(int i = 0;i<jdbcTypeFields.length;i++){


           //注意字段不要java的date类型，否则存不上数据库
            //4是数据库字段类型的编码
           jdbcTypeFields[i] = new JdbcTypeField(4
                   ,"fieldName"
                   ,Integer.class
                   ,"fieldName");
           fieldMap.put("fieldName","Integer");
       }
//
       //构造jdbc字段映射
       JdbcType jdbcType = new JdbcType();
       jdbcType.setCacheName("person");
       jdbcType.setKeyType(Integer.class);
       jdbcType.setValueType(Person.class);
//        jdbcType.setDatabaseSchema(SQL_SCHEMA);
       jdbcType.setDatabaseTable("person");
       jdbcType.setKeyFields(new JdbcTypeField(4
                   ,"fieldName"
                   ,Integer.class
                   ,"fieldName");


       jdbcType.setValueFields(jdbcTypeFields);
       storeFactory.setTypes(jdbcType);
        ccf.setBackups(1);
       ccf.setCacheStoreFactory(storeFactory);
        //会在ignite自动建表，并对持久化支持
       QueryEntity queryEntity = new QueryEntity();
       queryEntity.setTableName(tableName);
       queryEntity.setKeyType("java.lang.Integer");
       queryEntity.setValueType("model.person");
       queryEntity.setKeyFieldName("id");
       queryEntity.setKeyFields(Collections.singleton("id"));
       queryEntity.setFields(fieldMap);
       ccf.setQueryEntities(Collections.singleton(queryEntity));
       ignite.getOrCreateCache(ccf);
```

然后就可以使用构建的缓存进行操作啦

```java
            //执行select语句查询并获取返回结果
            Map<String, Object> result = new HashMap<>();
            // Getting a reference to an underlying cache created for table.
            IgniteCache<Integer, Person> cache = igniteEngine.getOrCreateCacheByCacheName("person");
            // Querying data from the cluster using a distributed JOIN.
            FieldsQueryCursor<List<?>> cursor = cache.query(new SqlFieldsQuery("select * from person").setSchema("public"));
            Iterator<List<?>> iterator = cursor.iterator();
            while (iterator.hasNext()) {
                List<?> row = iterator.next();
                int i = 0;
                for (; i < cursor.getColumnsCount(); i++) {
                    result.put(cursor.getFieldName(i).toLowerCase(), row.get(i));
                }
            }

            // 执行kv操作
            IgniteCache<Integer, Person> cache = igniteEngine.getOrCreateCacheByCacheName("person");
            Person person = cache.get(1);
            person.setName("hhbbz");
            cache.put(person);

            //执行insert/update语句
            IgniteCache<String, Map<String, Object>> cache = igniteEngine.getOrCreateCacheByCacheName("person"));
            //执行sql
            cache.query(new SqlFieldsQuery(builder.toString()).setSchema("public")).getAll();
```


# 后写缓存持久化的坑

## 后写缓存是什么

在一个简单的通写模式中每个缓存的put和remove操作都会涉及一个持久化存储的请求，因此整个缓存更新的持续时间可能是相对比较长的。另外，密集的缓存更新频率也会导致非常高的存储负载。

对于这种情况，Ignite提供了一个选项来执行异步化的持久化存储更新，也叫做后写，这个方式的主要概念是累加更新操作然后作为一个批量操作异步化地刷入持久化存储中。真实的数据持久化可以被基于时间的事件触发（数据输入的最大时间受到队列的限制），也可以被队列的大小触发（当队列大小达到一个限值），或者通过两者的组合触发，这时任何事件都会触发刷新。

>更新顺序

>对于后写的方式只有数据的最后一次更新会被写入底层存储。如果键为key1的缓存数据分别依次地更新为值value1、value2和value3，那么只有(key1,value3)对这一个存储请求会被传播到持久化存储。

>更新性能

>批量的存储操作通常比按顺序的单一存储操作更有效率，因此可以通过开启后写模式的批量操作来利用这个特性。简单类型（put和remove）的简单顺序更新操作可以被组合成一个批量操作。比如，连续地往缓存中加入(key1,value1),(key2,value2),(key3,value3)可以通过一个单一的CacheStore.putAll(...)操作批量处理。

## 这也是我遇到的第四个坑：

交代一下背景：

我整个项目中全部使用sql去对ignite进行缓存操作。当我使用CacheJdbcPojoStoreFactory做第三方存储持久化的时候，在大批量数据操作时，持久化总是会卡顿，很慢，并且会阻塞正常的内存操作。
虽然官方文档说后写缓存是异步化的持久化存储更新，但是我也没懂为什么会阻塞我正常的内存操作。

于是研究了几天后，经过各种猜测和反复实践，笔者所猜测的原因是：

>在持久化过程中，涉及到更新操作时候，ignite会在第三方存储表中先进行select，再进行update，这样的话，在大批量数据操作时，会造成大量的数据库死锁从而导致性能下降，使得阻塞持久化过程，这是异步持久化的阻塞。

那么为什么会阻塞正常的内存操作的？

笔者所猜测的原因是：

>当上一次持久化的数据还没全部插入到库表中时，就会阻塞ignite的内存操作了。举个例子就是，比如上一轮持久化的数据中，有“19岁”这个数据，这时候“19岁”这一条数据只存在内存中，还没保存到库表里，因为后写是批量的，假如我后写的batchsize是1000，那么这时候就已经构造好了1000条数据的批量insert/update语句，如果刚好“19岁”这条数据就在这1000条构造好update语句的数据里面，此时又在内存中把“19岁”update成“20岁”，并且再次触发持久化，那就得等待后写的批量轮询逻辑把“19岁”这个数据持久化到库表中为止，也因为这样，刚好就能解释为什么有些持久化update很快，有些却会卡住，都是取决于持久化update的那条数据是不是处于后写的batchsize的批量数据中。

当然也可能是上面说的更新顺序的原因。

# 总结

总的来说，我是对ignite的持久化不太满意的，也有可能是因为我实践不当吧，如果有什么错误的操作或者判断，欢迎邮箱指正。