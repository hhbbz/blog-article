---
title: Zookeeper和Etcd的对比
date: 2020-10-24 16:02:52
categories: 
- 后端
tags:
- Java
- 分布式
---

# 简介

Zookeeper 和 Etcd 都是非常优秀的分布式协调系统，zookeeper 起源于 Hadoop 生态系统，起步比较早，而 etcd 算是后起之秀，它的流行是因为它是 kubernetes 的后台支撑。

# Zookeeper

## 概述

zookeeper 起源于 Hadoop，后来进化为 Apache 的顶级项目。现在已经被广泛使用在 Apache 的项目中，例如 Hadoop，kafka，solr 等等。

{% asset_img 1.png 图片 %}

zookeeper 使用 ZAB 协议作为其一致性协议。 zookeeper 通过团队的形式工作，一组 node 一起工作，来提供分布式能力，这组 node 的数量需要是奇数。

第一个节点与其他节点沟通，选举出一个 leader，获取多数票数的成为 leader，这就是为什么需要奇数个 node，其他节点被称为follower。

client 连接 zookeeper 时可以连接任何一个，client 的读请求可以被任何一个节点处理，写请求只能被 leader 处理。所以，添加新节点可以提高读的速度，但不会提高写的速度。

对于 CAP 模型，zookeeper 保障的是 CP。

## ZNode

{% asset_img 2.png 图片 %}

存储数据时，zookeeper 使用树形结构，其中的每个节点称作 ZNode，访问一个 ZNode 时，需要提供从 root 开始的绝对路径。

每个 ZNode 可以存储最多 1MB 的数据，用户可以：

- 创建 ZNode
- 删除 ZNode
- 存储数据到指定 ZNode
- 从 ZNode 中读取数据

zookeeper 还提供了一个非常重要的特性：watcher API。

### zookeeper watches

用户可以对一个 ZNode 设置 watch，当这个 ZNode 发生了变化时，例如 创建、删除、数据变更、添加或移除子节点，watch API 就会发出通知，这是 zookeeper 非常重要的功能。

zookeeper 的 watch 有一个缺点，就是这个 watch 只能被触发一次，一旦发出了通知，如果还想对这个节点继续 watch，用户需要重新设置 watch。

## 优点

- 非阻塞全部快照（达成最终一致）
- 高效的内存管理
- 高可靠
- API 简单
- 连接管理可以自动重试
- ZooKeeper recipes 的实现是经过完整良好的测试的。
- 有一套框架使得写新的 ZooKeeper recipes 非常简单。
- 支持监听事件
- 发生网络分区时，各个区都会开始选举 leader，那么节点数少的那个分区将会停止运行

## 缺点

- zookeeper 是 java 写的，那么自然就会继承 java 的缺点，例如 GC 暂停。
- 如果开启了快照，数据会写入磁盘，此时 zookeeper 的读写操作会有一个暂时的停顿。
- 对于每个 watch 请求，zookeeper 都会打开一个新的 socket 连接，这样 zookeeper 就需要实时管理很多 socket 连接，比较复杂。

# Etcd

{% asset_img 3.png 图片 %}

## 概述

etcd 是用 go 开发的，出现的时间并不长，不像 zookeeper 那么悠久和有名，但是前景非常好。

etcd 是因为 kubernetes 而被人熟知的，kubernetes 的 kube master 使用 etcd 作为分布式存储获取分布式锁，这为 etcd 的强大做了背书。

{% asset_img 4.png 图片 %}

etcd 使用 RAFT 算法实现的一致性，比 zookeeper 的 ZAB 算法更简单。

etcd 没有使用 zookeeper 的树形结构，而是提供了一个分布式的 key-value 存储。

特性：

- 原子性
- 一致性
- 顺序一致性
- 可串行化级别
- 高可用
- 可线性化

## API

etcd3 提供了如下操作接口：

- put - 添加一个新的 key-value 到存储中
- get - 获取一个 key 的 value
- range - 获取一个范围的 key 的 value，例如：key1 - key10
- transaction - 读、对比、修改、写的组合
- watch - 监控一个或一个范围的 key，发生变化后就会得到通知

## 优点

{% asset_img 5.png 图片 %}

- 支持增量快照，避免了 zookeeper 的快照暂停问题
- 堆外存储，没有垃圾回收暂停问题
- 无需像 zookeeper 那样为每个 watch 都做个 socket 连接，可以复用
- zookeeper 每个 watch 只能收到一次事件通知，etcd 可以持续监控，在一次 watch 触发之后无需再次设置一次 watch
- zookeeper 会丢弃事件，etcd3 持有一个事件窗口，在 client 断开连接后不会丢失所有事件

## 缺点

- 如果超时，或者 client 与 etcd 网络中断，client 不会明确的知道当前操作的状态
- 在 leader 选举时，etcd 会放弃操作，并且不会给 client 发送放弃响应
- 在网络分区时，当 leader 处于小分区时，读请求会继续被处理

# 总结

zookeeper 是用 java 开发的，被 Apache 很多项目采用。

etcd 是用 go 开发的，主要是被 Kubernetes 采用。

zookeeper 非常稳定，是一个著名的分布式协调系统，etcd 是后起之秀，前景广阔。

因为 etcd 是用 go 写的，现在还没有很好的 java 客户端库，需要通过 http 方式调用。

而 zookeeper 在这方面就成熟很多，对于 java 之外的其他开发语言都有很好的客户端库。

具体选择 zookeeper 还是 etcd，需要根据您的需求结合它们各自的特性进行判断，还有您所使用的开发语言。

翻译整理自：
摘抄于 [Medium](https://medium.com/@Imesha94/apache-curator-vs-etcd3-9c1362600b26)

