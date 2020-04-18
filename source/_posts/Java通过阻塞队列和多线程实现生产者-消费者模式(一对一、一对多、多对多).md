---
title: Java通过阻塞队列和多线程实现生产者-消费者模式(一对一、一对多、多对多)
date: 2020-02-10 22:03:34
categories: 
- 后端
tags:
- Java
---

# 生产者-消费者模式是什么

- **生产者消费者模式是通过一个容器来解决生产者和消费者的强耦合问题**。生产者和消费者彼此之间不直接通讯，而通过阻塞队列来进行通讯，所以生产者生产完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列里取，阻塞队列就相当于一个缓冲区，平衡了生产者和消费者的处理能力。
- 这个阻塞队列就是用来给生产者和消费者解耦的。纵观大多数设计模式，都会找一个第三者出来进行解耦，如工厂模式的第三者是工厂类，模板模式的第三者是模板类。

# 为什么要使用生产者-消费者模式

- 在线程世界里，生产者就是生产数据的线程，消费者就是消费数据的线程。

- 在多线程开发当中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。为了解决这个问题于是引入了生产者和消费者模式。

# 阻塞队列BlockingQueue的介绍

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

## BlockingQueue的主要几种实现

- ArrayBlockingQueue：基于数组实现的一个阻塞队列，在创建ArrayBlockingQueue对象时必须制定容量大小。并且可以指定公平性与非公平性，默认情况下为非公平的，即不保证等待时间最长的队列最优先能够访问队列。

- LinkedBlockingQueue：基于链表实现的一个阻塞队列，在创建LinkedBlockingQueue对象时如果不指定容量大小，则默认大小为Integer.MAX_VALUE。
  
- PriorityBlockingQueue：以上2种队列都是先进先出队列，而PriorityBlockingQueue却不是，它会按照元素的优先级对元素进行排序，按照优先级顺序出队，每次出队的元素都是优先级最高的元素。注意，此阻塞队列为无界阻塞队列，即容量没有上限（通过源码就可以知道，它没有容器满的信号标志），前面2种都是有界队列。

# 通过简单的几个线程类实现

## 创建生产者类

```java
/**
 * @author hhbbz on 2020-02-10.
 * @Explain:
 */
public class Producer implements Runnable{
    private BlockingQueue<Integer> queue;
    public Producer(BlockingQueue queue) {
        this.queue = queue;
    }
    @Override
    public void run() {
        queue.offer(new Random().nextInt(100));
    }
}
```

## 创建消费者类

```java
/**
 * @author hhbbz on 2020-02-10.
 * @Explain:
 */
public class Consumer implements Runnable{
    private BlockingQueue<Integer> queue;
    public Consumer(BlockingQueue queue) {
        this.queue = queue;
    }
​
    @Override
    public void run() {
        Integer value = queue.poll();
    }
}
```

## 测试入口类

```java
/**
 * @author hhbbz on 2020-02-10.
 * @Explain:
 */
public class Main {
    public static void main(String[] args){
        //多个队列
        BlockingQueue<Integer> queue = new LinkedBlockingQueue<>();
        BlockingQueue<Integer> queue2 = new LinkedBlockingQueue<>();
​
        //多个生产者
        Thread producer1 = new Thread(new Producer(queue));
        Thread producer2 = new Thread(new Producer(queue));
        Thread producer3 = new Thread(new Producer(queue2));
        Thread producer4 = new Thread(new Producer(queue2));
        producer1.start();
        producer2.start();
        producer3.start();
        producer4.start();
​
        //多个消费者
        Thread consumer1 = new Thread(new Consumer(queue));
        Thread consumer2 = new Thread(new Consumer(queue));
        Thread consumer3 = new Thread(new Consumer(queue2));
        Thread consumer4 = new Thread(new Consumer(queue));
        Thread consumer5 = new Thread(new Consumer(queue2));
        Thread consumer6 = new Thread(new Consumer(queue));
        consumer1.start();
        consumer2.start();
        consumer3.start();
        consumer4.start();
        consumer5.start();
        consumer6.start();
    }
}
```

自己想怎么处理生产者、消费者和队列之间的关系，都能很直观的进行调整。

接下来列一下项目中常用到的实现方式。

# 通过线程池封装起来的实现代码（！最重要最重要最重要！）


## 创建队列服务配置启动类，包含生产消息，可按需拆解

```java
/**
 * @author hhbbz on 2020-02-10.
 * @Explain: 队列服务配置启动类，包含生产消息，可按需拆解
 */
@Component
@Slf4j
public class RecordQueueService {
    /**执行状态 */
    protected boolean isRunning;
    /**队列消费线程池 */
    private ThreadPoolExecutor executorService;
    //队列数量
    Integer queueNumber = 5;
    //队列长度
    Integer queueCapacity = 500;
    //每个队列对应多少个消费线程
    Integer singleQueueThreadNumber = 2;
    /**队列组列表 */
    List<BlockingQueue<Integer>> queueList = new ArrayList<>();
    //总线程数量，所有生产线程和消费线程
    Integer threadSize = queueNumber*singleQueueThreadNumber;

    public void start(String srvPoolName) {
        log.info("队列服务启动.......");
        ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("consume-"+srvPoolName+"-%d").build();

        //生产端线程和队列一对一
        for (int i = 0; i < queueNumber; i++) {
            queueList.add(new ArrayBlockingQueue<>(queueCapacity));
        }
        executorService = new ThreadPoolExecutor(
                threadSize, //线程池核心线程，至少要可以放入所有的生产线程和消费线程
                threadSize, //线程池容量大小
                300,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(threadSize+1),
                threadFactory
        );



        for (int i = 0; i < threadSize; i++) {
            //消费端
            //因为生产线程和队列一对一，通过getQueue取余的方式取到队列，即可实现多个消费线程消费同个队列
            executorService.submit(new SimpleRecordQueueHandler(this.getQueue(i),i));
        }
    }

    /**
     * 生产消息
     * @param str
     * @return
     */
    public Integer publish(String str) {
        if(!isRunning){
            //
        }
        BlockingQueue<Integer> queue = this.getQueue(str);
        try {
            if(queue!=null){
                queue.put(Integer.parseInt(str));
            }
        } catch (Exception e) {

        }
        if(queue!=null){
            return queue.size();
        }else{
            return 0;
        }

    }
    /**
     * 基于key值的hash值放在不同的队列里面
     * @param keyValue
     * @return
     */
    public BlockingQueue<Integer> getQueue(String keyValue){
        int p = keyValue.hashCode() % queueNumber;
        p = Math.abs(p);
        return getQueue(p);
    }

    //每个消费者对应的队列
    public BlockingQueue<Integer> getQueue(int position){
        position = position % queueList.size();
        if(position >= queueNumber || position <0){
            return queueList.get(0);
        }
        return queueList.get(position);
    }
}
```

## 创建消费类

```java
/**
 * @author hhbbz on 2020-02-10.
 * @Explain: 消费消息类
 */
@Slf4j
public class SimpleRecordQueueHandler implements Runnable {

    private static final Logger logger = LoggerFactory.getLogger(SimpleRecordQueueHandler.class);

    //队列内容
    private Queue<Integer> data;

    //队列编号
    private int handlerNumber;

    public SimpleRecordQueueHandler(Queue<Integer> data, int handlerNumber) {
        this.data = data;
        this.handlerNumber = handlerNumber;
    }

    /**
     * 详细的业务逻辑处理
     */
    @Override
    public void run() {
        logger.info("当前消费队列编号:{}",handlerNumber);
        Integer value = data.poll();
        //TODO 消费逻辑
    }
}
```

# 总结

最后一种实现方式是较为常用的，建议加深印象多理解理解，生产者-消费者模式在实践中非常广泛和实用，灵活配置一对一，一对多更是可以画龙点睛。


