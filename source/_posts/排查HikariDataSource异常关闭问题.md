---
title: 排查HikariDataSource异常关闭问题
date: 2019-12-09 16:48:02
categories: 
- 后端
tags:
- Spring
- Java
---

# Hikari简单介绍

[官网地址](https://github.com/brettwooldridge/HikariCP)

快速，简单，可靠的数据源，spring boot2.0 已经将 HikariCP 做为了默认的数据源链接池，在官网测试中秒杀一切其他数据源，比如 commons-dbcp,tomcat,c3po,druid。

## 基本设计

Hikari 链接池采用了很多优化来提高并发数，可参考[这里](https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole)
所有数据库链接池都遵守基本的设计规则，实现 javax.sql.DataSource 接口，里面最重要的方法就是 Connection getConnection () throws SQLException; 用于获取一个 Connection， 一个 Connection 就是一个数据库链接，就是一个 TCP 链接，建立 TCP 链接是需要进行 3 次握手的，这降低来链接的使用效率，也是各种数据库链接池存在的原因。

数据库链接池通过事先建立好 Connection 并缓存起来，这样应用需要做数据查询的时候，直接从缓存中拿到 Connection 就可以使用来。数据库链接池还能够检测异常的链接，释放闲置的链接。

# HikariDataSource

Hikari 中提供的 DataSource 是 HikariDataSource ，HikariDataSource 实现了 HikariConfig，和数据库的各种参数超时时间配置就正 HikariaConfig 中。

其中提供两种初始化方式，一种是默认的构造函数，单 new 一个 HikariDataSource 时，数据源的链接不会建立，需要等到第一次调用 HikariDataSource 的 getConnection 方法。数据源建立后的相关信息保存在 HikariDataSource 中变量 HikariPool pool。

另一种建立方式是调用带有 HikariConfig 的构造函数，这种方式适合多个数据源的建立，共享同一份配置。 这种方式在调用构造函数的时候就建立了数据源的链接。

HikariDataSource 的所有数据源获取都委托给了 HikariPool，一个数据源会有一个 HikariPool，一个 HikariPool 中有一个 ConcurrentBag，一个 ConcurrentBag 中多个 PoolEntry，一个 PoolEntry 对应一个 Connection。

# 问题描述

前段时间做的一个数据平台在最近总是莫名其妙出现 
```java
Caused by:
java.sql.SQLException: HikariDataSource HikariDataSource (HikariPool-2) has been closed.
```
这种错误。按字面意思来说就是使用连接池里的连接执行sql之前，这个连接池就已经被关闭了无法使用。

因为是会有很多数据源配置通过数据平台来连接数据库的，连接数非常庞大，很难管理，也会容易出现这种因为遗漏逻辑代码而导致的问题。所以在处理大量的数据库连接时候，务必要仔细仔细再仔细。**“连接风暴”**这种事情一旦发生，会压倒整个数据库服务器，直接影响当前业务，甚至所有业务。

# 问题跟踪

在找问题的时候，我们要看清楚控制台输出的堆栈信息和调用链，除了关注自己本身的业务代码之外，也要去关注调用链经过的框架/工具。

因为这个问题会涉及到数据库方面，所以数据库那一边的参数以及资源情况我们也要去关注，看到底是程序连接泄漏还是数据库给的连接数实在是太少。

# 模拟场景确认问题

在通过报错信息，异常产生的调用链以及所涉及资源的使用情况、参数配置之后，就要开始根据自己的猜想去编写代码，模拟bug触发的场景去确认我们的问题了。


## 封装一个简单的HikariDataSource
  
```java
    public HikariDataSource initDataSource(){
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setConnectionTestQuery("SELECT 1");
        dataSource.setConnectionTimeout(60000);
        dataSource.setMinimumIdle(2);
        dataSource.setMaximumPoolSize(100);
        dataSource.setMaxLifetime(600000);
        dataSource.setValidationTimeout(5000);
        dataSource.setIdleTimeout(300000);
        dataSource.setLeakDetectionThreshold(500000);

        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/anymarket_dev?useSSL=false&characterEncoding=utf8&allowPublicKeyRetrieval=true");
        dataSource.setUsername("root");
        dataSource.setPassword("hbz961005");
        return dataSource;
    }
```

## 使用构造的数据源进行查询
  
```java
    @Test
    public void testHikari() throws InterruptedException {
        HikariDataSource dataSource = initDataSource();

        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);

        jdbcTemplate.execute("select * from clocd_type");

    }
```

这样的操作是正常使用的，没有问题，但是我们的报错有异常关闭数据源连接，所以我们下一步就要开始加一句close数据源，再执行看看。

## 基于上面的测试代码进行改造，增加关闭数据源代码。

```java
    @Test
    public void testHikari() throws InterruptedException {
        HikariDataSource dataSource = initDataSource();

        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);

        jdbcTemplate.execute("select * from clocd_type");

        dataSource.close();

        jdbcTemplate.execute("select * from clocd_type");
    }
```

执行，这个时候控制台就报出了
```java
Caused by:
java.sql.SQLException: HikariDataSource HikariDataSource (HikariPool-2) has been closed.
```
这跟我们数据平台遇到的错误是一样的。

自此，我们已经验证出来了导致问题的所在：
    jdbcTemplate是基于初始化出来的dataSource构造出来的，那么当我们的dataSoource关闭之后，
    我们再使用原先的jdbcTemplate去进行查询，那就会有问题，因为里面的连接已经被我们关闭了。

问题已经验证出来了，那么接下来就是解决问题。

## 解决代码

基于上面的测试代码，补上重新构造新的datasource代码，重新设置当前jdbcTemplate。

```java
    @Test
    public void testHikari() throws InterruptedException {
        HikariDataSource dataSource = initDataSource();

        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);

        jdbcTemplate.execute("select * from clocd_type");

        dataSource.close();

        dataSource = initDataSource();

        jdbcTemplate = new JdbcTemplate(dataSource);

        jdbcTemplate.execute("select * from clocd_type");
    }
```

这样就不会再报错了。

# 把问题回溯到业务代码中

我们已经知道了这个报错产生的原因以及解决方式了，那么我们就要去检查我们的业务代码。很明显业务代码报这个错的原因就是关闭连接后，没有重新构造连接，并设置给jdbcTemplate，所以只需要在业务代码中补上这个解决的逻辑即可。

# 结尾

这篇文章主要说明从一个复杂的业务平台中，把异常抽离出来，并使用简单的测试类/单元测试去进行模拟场景去验证问题，验证问题的产生原因以及找到解决方法，并最后把解决思路回归到业务代码中进行检查。

这种报错还是比较简单的，很好验证。如果是遇到因为线程安全、批量并发和中间件等所导致的bug，那么我们大概率要借助工具去进行问题跟踪了。这里笔者推几个工具:

1. alibaba arthas
2. jmeter
3. qunar bistoury

这些都是挺不错的bug跟踪验证工具，大家可以去了解一下。
