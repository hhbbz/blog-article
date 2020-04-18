---
title: Redis分布式锁及优化
date: 2018-01-17 22:59:05
updated: 2018-01-17 23:14:22
categories: 
- 后端
tags:
- Redis
- Java
- 分布式
---
# 自身业务场景

场景一: 我有一个数据服务，每天调用量在3亿，每天按86400秒计算的qps在4000左右，由于服务的白天调用量要明显高于晚上，所以白天下午的峰值qps达到6000的，一共有4台服务器，单台qps要能达到3000以上。**我最终使用了redis的setnx()和expire()的分布式锁解决的问题。**

场景二:场景一变异版。在这个场景中，不涉及支付。但是由于资源分配一次过程中，需要保持涉及一致性的地方增加，而且一期的设计目标要达到峰值qps500，所以需要我们对场景进一步的优化。**我最终使用了redis的setnx()、expire()和基于数据库表的分布式锁来解决的问题。**  

# 分布式锁的解决方式

1. 首先明确一点，有人可能会问是否可以考虑采用ReentrantLock来实现，但是实际上去实现的时候是有问题的，ReentrantLock的lock和unlock要求必须是在同一线程进行，而分布式应用中，lock和unlock是两次不相关的请求，因此肯定不是同一线程，因此导致无法使用ReentrantLock。

2. 基于数据库表做乐观锁，用于分布式锁。

3. 使用memcached的add()方法，用于分布式锁。

4. 使用memcached的cas()方法，用于分布式锁。(不常用)

5. 使用redis的setnx()、expire()方法，用于分布式锁。

6. 使用redis的setnx()、get()、getset()方法，用于分布式锁。

7. 使用redis的watch、multi、exec命令，用于分布式锁。(不常用)

8. 使用zookeeper，用于分布式锁。(不常用)

这里着重讲Redis的分布式锁。

# 使用redis的setnx()、expire()方法，用于分布式锁

对于使用redis的setnx()、expire()来实现分布式锁，这个方案相对于memcached()的add()方案，redis占优势的是，其支持的数据类型更多，而memcached只支持String一种数据类型。除此之外，无论是从性能上来说，还是操作方便性来说，其实都没有太多的差异，完全看你的选择，比如公司中用哪个比较多，你就可以用哪个。

首先说明一下setnx()命令，setnx的含义就是SET if Not Exists，其主要有两个参数 setnx(key, value)。该方法是原子的，如果key不存在，则设置当前key成功，返回1；如果当前key已经存在，则设置当前key失败，返回0。但是要注意的是setnx命令不能设置key的超时时间，只能通过expire()来对key设置。

具体的使用步骤如下:

1. setnx(lockkey, 1)  如果返回0，则说明占位失败；如果返回1，则说明占位成功

2. expire()命令对lockkey设置超时时间，为的是避免死锁问题。

3. 执行完业务代码后，可以通过delete命令删除key。

这个方案其实是可以解决日常工作中的需求的，但从技术方案的探讨上来说，可能还有一些可以完善的地方。比如，如果在第一步setnx执行成功后，在expire()命令执行成功前，发生了宕机的现象，那么就依然会出现死锁的问题，所以如果要对其进行完善的话，可以使用redis的setnx()、get()和getset()方法来实现分布式锁。

# 使用redis的setnx()、get()、getset()方法，用于分布式锁

这个方案的背景主要是在setnx()和expire()的方案上针对可能存在的死锁问题，做了一版优化。

那么先说明一下这三个命令，对于setnx()和get()这两个命令，相信不用再多说什么。那么getset()命令？这个命令主要有两个参数 getset(key，newValue)。该方法是原子的，对key设置newValue这个值，并且返回key原来的旧值。假设key原来是不存在的，那么多次执行这个命令，会出现下边的效果：

1. getset(key, "value1")  返回nil   此时key的值会被设置为value1

2. getset(key, "value2")  返回value1   此时key的值会被设置为value2

3. 依次类推！

介绍完要使用的命令后，具体的使用步骤如下：

1. setnx(lockkey, 当前时间+过期超时时间) ，如果返回1，则获取锁成功；如果返回0则没有获取到锁，转向2。

2. get(lockkey)获取值oldExpireTime ，并将这个value值与当前的系统时间进行比较，如果小于当前系统时间，则认为这个锁已经超时，可以允许别的请求重新获取，转向3。

3. 计算newExpireTime=当前时间+过期超时时间，然后getset(lockkey, newExpireTime) 会返回当前lockkey的值currentExpireTime。

4. 判断currentExpireTime与oldExpireTime 是否相等，如果相等，说明当前getset设置成功，获取到了锁。如果不相等，说明这个锁又被别的请求获取走了，那么当前请求可以直接返回失败，或者继续重试。

5. 在获取到锁之后，当前线程可以开始自己的业务处理，当处理完毕后，比较自己的处理时间和对于锁设置的超时时间，如果小于锁设置的超时时间，则直接执行delete释放锁；如果大于锁设置的超时时间，则不需要再锁进行处理。

但这套方案也会存在一些质疑点：

1. 在“get(lockkey)获取值oldExpireTime ”这个操作与“getset(lockkey, newExpireTime) ”这个操作之间，如果有N个线程在get操作获取到相同的oldExpireTime后，然后都去getset，会不会返回的newExpireTime都是一样的，都会是成功，进而都获取到锁？

>我认为这套方案是不存在这个问题的。依据有两条: 第一，redis是单进程单线程模式，串行执行命令。 第二，在串行执行的前提条件下，getset之后会比较返回的currentExpireTime与oldExpireTime 是否相等。

2. 在“get(lockkey)获取值oldExpireTime ”这个操作与“getset(lockkey, newExpireTime) ”这个操作之间，如果有N个线程在get操作获取到相同的oldExpireTime后，然后都去getset，假设第1个线程获取锁成功，其他锁获取失败，但是获取锁失败的线程它发起的getset命令确实执行了，这样会不会造成第一个获取锁的线程设置的锁超时时间一直在延长？

>我认为这套方案确实存在这个问题的可能。但我个人认为这个微小的误差是可以忽略的，不过技术方案上存在缺陷，大家可以自行抉择。