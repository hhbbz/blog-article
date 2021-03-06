---
title: ignite实践踩坑录(3) 数据平衡分布
date: 2019-09-23 14:49:44
categories: 
- 后端
tags:
- Ignite
- Java
- 分布式
---

# 为什么需要数据平衡分布

在一个集群的环境中，我们往往希望能更好的利用分布式的优势，把数据平衡分布在各个节点中，这样有几个好处：

1. 这样才能缓存/持久化更多的数据。

2. 即使其中一个节点down掉，也不影响其他节点的数据和状态。

3. 数据在各个节点中平衡分布，使得资源利用率，单节点的拔插效率更高。

# igntie实现分布式存储的配置

Ignite提供了三种不同的缓存操作模式，```分区```、```复制```和```本地```。缓存模型可以为每个缓存单独配置，缓存模型是通过```CacheMode```枚举定义的。

## 分区模式

```分区```模式是扩展性最好的分布式缓存模式，这种模式下，所有数据被均等地分布在分区中，所有的分区也被均等地拆分在相关的节点中，实际上就是为缓存的数据创建了一个巨大的```内存内分布式存储```。这个方式可以在所有节点上只要匹配总可用存储(内存和磁盘)就可以存储尽可能多的数据，因此，可以在集群的所有节点的内存中可以存储TB级的数据。也就是说，只要有足够多的节点，就可以存储足够多的数据。

与```复制```模式不同，它更新是很昂贵的，因为集群内的每个节点都需要更新，而分区模式更新就很廉价，因为对于每个键只需要更新一个主节点（可选择一个或者多个备份节点），不过读取变得较为昂贵，因为只有特定节点才持有缓存的数据。

为了避免额外的数据移动，总是访问恰好缓存有要访问的数据的节点是很重要的，这个方法叫做```关联并置```，当工作在分区化缓存时强烈建议使用。

    分区化缓存适合于数据量很大而更新频繁的场合。

## 基线拓扑

基线拓扑是一组Ignite服务端节点，目的是同时在内存以及原生持久化中存储数据。基线拓扑中的节点在功能方面不受限制，并且作为数据和计算的容器，在行为上也和普通的服务端节点一样。可以实现在各个节点的磁盘中平衡存放持久化数据。

在开启持久化之后，集群默认未激活，因此需要先激活集群，再对基线拓扑进行操作。

## 注意事项

* 基线拓扑是实现分布式存储持久化数据的必要，而partition的缓存模式只是决定了内存数据在各个节点中的分布存放。
* 第一次启动集群时，要等所有节点都单独启动完成之后，再激活集群，才能把集群中所有节点加入到基线拓扑中。
* 可以通过```control.sh```来堆基线拓扑进行操作，添加节点/删除节点。也可通过代码来进行操作:
  
>>
    Ignite ignite = Ignition.start();

    ignite.cluster().active(true);

    Collection<ClusterNode> nodes = ignite.cluster().forServers().nodes();

    ignite.cluster().setBaselineTopology(nodes);

# ignite数据平衡分布的实践场景

内存缓存模式```CacheMode```使用```分区```，保证数据在各个节点的内存中缓存起来。

* 第一次环境部署，所有节点要单独启动，所有节点启动完成之后调用激活集群接口，ignite自动将所有节点集群，并且初始化基线拓扑，把所有节点都在基线拓扑变为上线状态。

* 集群中所有节点挂掉，即所有节点在基线拓扑中处于下线状态，这时候启动其中一个节点会自动从原先的基线拓扑中变为上线状态，后续其他节点也是，然后触发数据平衡。

* 集群中单个节点挂掉，即在基线拓扑中处于下线状态，那直接启动这个节点就行，会自动从原先的基线拓扑中上线然后数据平衡

* 要把新节点加入到已激活的集群，并且加入到集群中的基线拓扑中的话，要调用激活集群接口（接口ip是当前节点或者是集群中任意一个节点的都可以），或者使用```./control.sh --baseline add```命令加入集群的基线拓扑后，集群会把部分数据分布到新节点中去

* 要把下线节点从集群中删去，并确认不再需要这个节点，也不需要这个节点里面的数据，则把当前节点shutdown掉，然后重新调用激活集群接口即可，同时该节点也无法持有旧数据重新加入集群中。（危险操作）

# 日志示例

## 加入新节点前

{% asset_img 001.png 图片 %}

## 加入新节点

{% asset_img 002.png 图片 %}

## 加入新节点后，尚未激活的集群

{% asset_img 003.png 图片 %}