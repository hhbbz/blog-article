---
title: ignite踩坑录(4) 数据一致性
date: 2020-03-21 14:53:08
categories: 
- 后端
tags:
- Ignite
- Java
- 分布式
---

# 数据一致性的简要说明

说起一致性，我们大概第一反应就是想起分布式系统的CAP定理以及其相关定义，简单来说就是，一致性、可用性和分布式，三者之间最多只可保证两者的完整性。那有了这个思想前提，就可以来聊一聊ignite在数据一致性上面的部分设计。

# 问题场景说明
跑120w数据，在ignite里面的逻辑是读取localNode的数据，去put到一个集群缓存中。然后我有三个节点，当我对这三个节点个字执行一次这种操作之后，发现落盘的数据总是缺个十多条几十条。我业务逻辑中有日志记录，证实了业务代码的处理是没丢数据的。

# 在ignite中保证一致性

ignite是一个高可用高性能的内存数据库，所以其默认策略也是优先保证高可用和高性能为主，然后才是一致性。为了保证这种策略的的运行，ignitem默认开启了从备份读数据，这样就是提高了性能，弱化了一致性的体现。恰巧主备的默认读写策略是异步主从双写，从备份读的，这样也就导致了如果你需要保证严格的一致性，就需要把从备份读关闭。

# 代码配置修正

针对上述情况，我们可以选择两种解决方式。

1. 把备份关掉

```java
        CacheConfiguration<String,E> cacheConfiguration = new CacheConfiguration<>();
        cacheConfiguration.setBackups(0);
```

2. 把从备份读取数据给关掉

```java
        CacheConfiguration<String,E> cacheConfiguration = new CacheConfiguration<>();
        cacheConfiguration.setBackups(1);
        //设置备份的情况下，该属性要设置成false，否则会出现数据不一致
        cacheConfiguration.setReadFromBackup(false);
```

# 特别备注说明一个纠缠很久的坑 Faild to read WAL record at position xx size xx

在ignite连续跑大批量数据，会与jvm做大量的堆内堆外数据交互，所以对jvm的参数的优化是必不可少的。如果因为业务代码效率跟不上，则会堵塞ignite的正常交互，然后会导致wal文件记录某个地方速度慢了，又继续写入了什么触发的，就会报错Faild to read WAL record at position xx size xx。

这里我建议大家都使用gridgain社区版的ignite版本，上面的问题我在开源版2.7.0中遇到，换成社区版8.7.12之后，情况好了非常多，可用性也提高了很多。

[更换ignite社区版本](https://www.gridgain.com/docs/latest/developers-guide/setup#using-maven)
