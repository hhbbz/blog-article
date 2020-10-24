---
title: ignite实践踩坑录(2) 节点集群
date: 2019-07-13 11:26:25
categories: 
- 后端
tags:
- Ignite
- Java
- 分布式
---

# Ignite的天然支持分布式

Ignite具有非常先进的集群能力，包括逻辑集群组和自动发现。
Ignite是一个以内存为中心的分布式数据库，通过DiscoverySpi节点可以彼此发现对方，而且提供了TcpDiscoverySpi作为DiscoverySpi的默认实现，它使用TCP/IP来作为节点发现的实现，可以配置成基于组播的或者基于静态IP的。
所以在同一个网络内的ignite节点都会天然的自动的互相发现。如下图所示

{% asset_img 001.png 图片 %}


>第一坑：如果服务器中无法使用root账户，或者不是使用root账号进行ignite的启动的话，ignite节点之间是无法自动互相发现的，这时候需要使用TcpDiscoveryVmIpFinder来指定url和端口，去进行集群处理。

```java
            IgniteConfiguration cfg = new IgniteConfiguration();
            TcpDiscoverySpi spi = new TcpDiscoverySpi();
            TcpDiscoveryVmIpFinder ipFinder = new TcpDiscoveryVmIpFinder();
            // Set initial IP addresses.
            // Note that you can optionally specify a port or a port range.
            ipFinder.setAddresses(igniteProperties().getClusterAddrList());
            spi.setIpFinder(ipFinder);
            // Override default discovery SPI.
            cfg.setDiscoverySpi(spi);
```

## Ignite节点之间的数据平衡

> 第二坑：通过上面TcpDiscoveryVmIpFinder这种指定集群的发现方式，我们会发现持久化的数据全部都存在了指定地址列表的节点中，
当前机器自身的节点是不会有持久化的数据保存进来的。
即数据无法平衡分布在各个节点中。

原因在官方文档中有写：
> TcpDiscoveryVmIpFinder默认用的是非共享模式，如果希望启动一个服务端节点，那么在该模式中的IP地址列表同时也要包含本地节点的一个IP地址。它允许节点不等待其它节点加入集群，而是成为第一个集群节点并正常运行

所以我们的解决方案是：给每一个集群节点都配上**基线拓扑**。

### 基线拓扑主要概念

1. 如果启用了原生持久化，Ignite引入了一个基线拓扑的概念，它表示集群中将数据持久化到磁盘的一组服务端节点。
2. 基线拓扑是一组Ignite服务端节点，目的是同时在内存以及原生持久化中存储数据。基线拓扑中的节点在功能方面不受限制，并且作为数据和计算的容器，在行为上也和普通的服务端节点一样。

简单来说就是，基线拓扑可以让集群中的每个节点都有持久化存储的能力。

配置基线拓扑的代码也非常简单：

```java
        Ignite ignite = Ignition.start();
        if(!ignite.cluster().active()){
            ignite.cluster().active(true);

        }
        //使用基线拓扑进行ignite集群
        Collection<ClusterNode> nodes = ignite.cluster().forServers().nodes();

        ignite.cluster().setBaselineTopology(nodes);
```

使用基线拓扑之后，持久化数据就能在集群的各个节点中平衡分布了。

# Ignite的集群部署方式

Ignite的部署模式非常的灵活，在实际的场景中可以针对实际需要采用不同的部署方式，下面做简单的总结和对比：

## 独立式Ignite集群

这种情况下，集群的部署完全独立于应用，这个集群可以用于分布式计算，分布式缓存，分布式服务等，这时应用以客户端模式接入集群进行相关的操作，大体是如下的部署模式：

{% asset_img 002.png 图片 %}

* 优点：对已有的应用运行环境影响小，并且这个集群可以共享，为多个应用提供服务，对整个应用来说，额外增加了很多的计算和负载能力。
* 缺点：需要单独的一组机器，相对成本要高些，如果缓存操作并发不高或者计算不饱和，存在资源利用率低的情况。整体架构也变得复杂，维护成本也要高些。

## 嵌入式Ignite集群

这种情况下，可以将必要的jar包嵌入已有应用的内部，利用Ignite的发现机制，自动建立集群，大体是如下的部署模式：

{% asset_img 003.png 图片 %}

* 优点：无需额外增加机器，成本最低，Ignite可以和应用无缝集成，所有节点都为服务端节点，可以充分利用Ignite的丰富功能。这个模式可扩展性最好，简单增加节点即可快速扩充整个系统的计算和负载能力。
* 缺点：Ignite占用了服务器的部分资源，对应用整体性能有影响，可能需要进行有针对性的优化，应用更新时，集群可能需要重启，这时如果Ignite需要加载大量的数据，重启的时间可能变长，甚至无法忍受。

## 混合式Ignite集群

这种情况下，将上述2种模式混合在一起，即同时增加机器部署独立集群，同时又将Ignite嵌入应用内部以服务端模式运行，通过逻辑集群组进行资源的分配，整体上形成更大的集群，大体是如下的部署模式：

{% asset_img 004.png 图片 %}

这种模式更为灵活，调优后能做到成本、功能、性能的平衡，综合效果最佳。这时可以将缓存的数据通过集群组部署到应用外部的节点上，这样可以避免频繁的冷启动导致缓存数据频繁的长时间加载，对于计算，也能够动态地充分利用所有计算节点的资源。

[Ignite的集群部署方式段落来源于](https://my.oschina.net/liyuj/blog/651036)
