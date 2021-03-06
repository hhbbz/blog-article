---
title: Mysql索引详解
date: 2017-11-28 21:29:08
updated: 2017-11-29 10:50:49
categories: 
- 后端
tags:
- Mysql
---

# 数据库索引

是数据库管理系统中一个排序的数据结构，以协助快速查询，更新数据库表中数据，索引的实现通常使用B树（所有节点的平衡因子均为0的多叉查找树）及其变种B+树 。B+树特点是,只有在最低节点保存数据本身。节点保存主键。

# 存储引擎比较

- InnoDB: 写多读少，支持事务，不加锁读取，支持外键，支持行锁（where对主键的支持，select count(*)和order by 还是会锁表），不支持全文索引，InnoDB把数据核索引存放在表空间里面

- MyISAM: 读多写少，不支持事务，不支持外键，支持全文搜索，count()查询更快，MyISAM索引和数据是分开的，并且索引是有压缩的，提高内存使用率，能加载更多索引

1. InnoDB 中不保存表的具体行数，也就是说，执行select count() from table时，InnoDB要扫描一遍整个表来计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count()语句包含 where条件时，两种表的操作是一样的。
2. 对于AUTO_INCREMENT类型的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引。
3. DELETE FROM table时，InnoDB不会重新建立表，而是一行一行的删除。
4. LOAD TABLE FROM MASTER操作对InnoDB是不起作用的，解决方法是首先把InnoDB表改成MyISAM表，导入数据后再改成InnoDB表，但是对于使用的额外的InnoDB特性(例如外键)的表不适用。

    另外，InnoDB表的行锁也不是绝对的，假如在执行一个SQL语句时MySQL不能确定要扫描的范围，InnoDB表同样会锁全表，例如update table set num=1 where name like “%aaa%”


# 索引在不同存储引擎中的比较

## InnoDB

{% asset_img 002.png 图片 %}

{% asset_img 003.png 图片 %}

## MyISAM

{% asset_img 001.png 图片 %}

# 数据库索引的原理

数据库索引，是数据库管理系统中一个排序的数据结构，以协助快速查询、更新数据库表中数据。索引的实现通常使用B树及其变种B+树。

* 为什么要用 B-tree

一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，相对于内存存取，I/O存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数。
* 聚集索引与非聚集索引的区别

1. 聚集索引一个表只能有一个，而非聚集索引一个表可以存在多个
2. 聚集索引存储记录是物理上连续存在，而非聚集索引是逻辑上的连续，物理存储并不连续
3. 聚集索引:物理存储按照索引排序；聚集索引是一种索引组织形式，索引的键值逻辑顺序决定了表数据行的物理存储顺序

   非聚集索引:物理存储不按照索引排序；非聚集索引则就是普通索引了，仅仅只是对数据列创建相应的索引，不影响整个表的物理存储顺序.

4. 索引是通过二叉树的数据结构来描述的，我们可以这么理解聚簇索引：索引的叶节点就是数据节点。而非聚簇索引的叶节点仍然是索引节点，只不过有一个指针指向对应的数据块。

# 索引的数据结构

在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。**一是增加了数据库的存储空间，二是在插入和修改数据时要花费较多的时间(因为索引也要随之变动)。**

{% asset_img 004.jpg 图片 %}

上图展示了一种可能的索引方式。左边是数据表，一共有两列七条记录，最左边的是数据记录的物理地址（注意逻辑上相邻的记录在磁盘上也并不是一定物理相邻的）。为了加快Col2的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引键值和一个指向对应数据记录物理地址的指针，这样就可以运用二叉查找在O(log2n)的复杂度内获取到相应数据。
创建索引可以大大提高系统的性能。

1. 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。
2. 可以大大加快数据的检索速度，这也是创建索引的最主要的原因。
3. 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。
4. 在使用分组和排序子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。
5. 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

# 为什么不对表中的每一个列创建一个索引呢

1. 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加。
2. 索引需要占物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大。
3. 当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。

## 应该在这些列上建立索引

1. 在经常需要搜索的列上，可以加快搜索的速度
2. 在作为主键的列上，强制该列的唯一性和组织表中的排列结构
3. 在经常用在连接的列上，这些列主要是一些外键，可以加快连接的速度
4. 在需要根据范围进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的。
5. 在经常需要排序的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询的时间。
6. 在经常使用where子句中的列上面创建索引，加快条件的判断速度。