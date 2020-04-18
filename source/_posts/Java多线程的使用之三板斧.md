---
title: Java多线程的使用之三板斧
date: 2019-12-06 15:31:50
categories: 
- 后端
tags:
- Java
---

# 场景

在我们实际开发过程中，往往会遇到执行接口逻辑以及批任务处理的的执行效率问题，在这些场景中，都可以通过使用多线程的方式，把占据长时间的程序中的任务放到后台去处理，更好的发挥计算机的多核cpu的优势。

# 概述
这篇文章只介绍开发过程中实用的多线程代码的三种编写方法和实践过程，参数说明、线程安全、多线程之间的调度策略和状态同步这里就不多介绍了，会在后面的文章中加以详细说明。

# 多线程三板斧

- **CompletableFuture** 配合 **TaskExecutor** 异步执行
- **ThreadFactory** 、 **TaskExecutor** 配合 service和handler的消费者模式 异步执行
- **ForkJoin**将任务切割成子任务，并行执行

## CompletableFuture 配合 TaskExecutor 异步执行

CompletableFuture是java8新增加的类，提供了非常强大的Future的扩展功能，可以帮助我们简化异步编程的复杂性，提供了函数式编程的能力。

下面是简单使用CompletableFuture进行异步任务的执行。

1. 封装一个**TaskExecutor**

```java
@Configuration
public class AsyncConfiguration {

    /**异步执行的线程池 */
    @Bean
    public TaskExecutor dataAsyncTaskExecutor(DataImportProperties importProperties){
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(importProperties.getThreadNumber());
        taskExecutor.setMaxPoolSize(50);
        taskExecutor.setThreadGroupName("data-async-importer");
        taskExecutor.setThreadNamePrefix("data-async");
        taskExecutor.initialize();
        return taskExecutor;
    }
}
```

2. 在上面封装的线程池中使用**CompletableFuture**

```java
    @Resource(name = "dataAsyncTaskExecutor")
    private TaskExecutor taskExecutor;
    //异步任务
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(()->1,taskExecutor);
    //阻塞异步任务获取结果
    future.get();
```

    如果使用CompletableFuture过程中不传入自己封装的线程池，CompletableFuture会使用ForkJoinPool.commonPool()，它是一个会被很多任务 共享 的线程池，比如同一 JVM 上的所有 CompletableFuture、并行 Stream 都将共享 commonPool，除此之外，应用代码也能使用它。

## **ThreadFactory** 、 **TaskExecutor** 配合 service和handler的消费者模式 异步执行

这种是大家比较常用的异步执行任务的做法。代码比较直观，更容易调试。下面展示一个**多线程异步批次消费队列**的实践代码。

1. 定义队列数据封装

```java
@Getter
@Setter
public class QueueData {

    /**队列数据 */
    private Map<String,Object> data;

    /**数据的数据标识 */
    private String dbTableCode;
}
```

2. 定义队列任务生产端

```java
import java.util.Map;
import java.util.Queue;
/**
 * 采用内存队列作为消息处理服务缓存
 */
public class MemoryQueueService {
    public Queue<QueueData> queue;

    public MemoryQueueService(Queue<QueueData> queue){
        this.queue = queue;
    }
    //推入队列
    public Integer publish(Map<String, Object> eventData) {
        QueueData queueData = new QueueData();
        queueData.setData(eventData);
        queue.offer(queueData);
        return queue.size();
    }
}
```

3. 定义队列任务批次消费端handler

```java
import lombok.extern.slf4j.Slf4j;

import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * 内存队列的消费端
 */
@Slf4j
public class MemoryQueueDataHandler implements Runnable {

    private int batchSize;

    private List<QueueData> eventCache;

    private Queue<QueueData> queue;
    /**
     * 是否运行
     */
    private boolean running;

    public MemoryQueueDataHandler(Queue<QueueData> queue, int batchSize) {
        this.queue = queue;
        this.batchSize = batchSize;
        eventCache = new ArrayList<>(batchSize);
        running = true;
    }

    @Override
    public void run() {
        log.info("内存队列数据监听handler启动.");
        QueueData eventData=null;
        while (running || eventData!=null) {
            //消费
            eventData = queue.poll();
            if (eventData != null) {
                //事件消息不为空
                eventCache.add(eventData);
                //批量写入
                if (eventCache.size() >= batchSize) {
                    flushCacheToDb();
                }
            } else if(!eventCache.isEmpty()){
                //缓存不为空
                flushCacheToDb();
            } else {
                //如果队列为空,缓存也为空则等待
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                }
            }
        }
        //刷新缓存
        flushCacheToDb();
    }
    private void flushCacheToDb(){
        Map<String,List<Map<String,Object>>> wData = new HashMap<>(batchSize);
        //构造批次
        for (QueueData queueData : eventCache) {
            List<Map<String, Object>> cacheQueue = wData.computeIfAbsent(queueData.getDbTableCode(), k -> new ArrayList<>(batchSize));
            cacheQueue.add(queueData.getData());
            wData.put(queueData.getDbTableCode(),cacheQueue);
        }
        //批量写入
        for (Map.Entry<String, List<Map<String, Object>>> entry : wData.entrySet()) {
            //TODO 写入逻辑
        }
        eventCache.clear();
    }
    public void setRunning(boolean running) {
        this.running = running;
    }
}
```

4. 简单定义**ThreadFactory**、消费端**ExecutorService**、生产端**ConcurrentLinkedQueue**,
   并新增任务。

```java
            private ExecutorService executorService;
            private MemoryQueueService queueService;
            ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("event-consume-%d").build();
            //后面把异步任务委托给ExecutorService
          executorService = new ThreadPoolExecutor(
                    3, //核心线程
                    5, //最大线程
                    300,
                    TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    threadFactory
            );
            //存放数据的队列
            ConcurrentLinkedQueue<QueueData> queue = new ConcurrentLinkedQueue<>();
            //生产端
            queueService = new MemoryQueueService(queue);
            //启用5个线程进行消费
            for (int i = 0; i < 5; i++) {
                //消费端
                executorService.submit(new MemoryQueueDataHandler(dataEngine,queue,dataProperties.getBatchSize()));
            }

        //往队列中缓存数据
        HashMap<String,Object> map = new HashMap();
        queueService.publish(map)
```

## **ForkJoin**将大任务切割成小任务，并行执行

支和并框架的目的是以递归的方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体的结果，它是ExecutorService的一个实现，它把子任务分配给线程池（ForkJoinPool）中的工作线程。

1. 基于ForkJoin封装拆分任务，执行逻辑的抽象类

```java
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.RecursiveTask;
import java.util.function.Function;

@Slf4j
public class ListForkJoinExecution<V, R> extends RecursiveTask<R> {

    /**
     * 待处理数据
     */
    private transient List<V> values;

    /**
     * 单元逻辑执行函数
     */
    private transient Function<V, R> function;

    /**
     * 结果队列
     */
    private transient ConcurrentLinkedQueue<ListForkJoinExecution<V, R>> resultQueue;

    public ListForkJoinExecution(List<V> values, Function<V, R> function){
        this.values = values;
        this.function = function;
    }

    public void setResult(ConcurrentLinkedQueue<ListForkJoinExecution<V, R>> resultQueue) {
        this.resultQueue = resultQueue;
    }

    @Override
    protected R compute() {
        int len = values.size();

        try {
            if(len >= 3){
                int min = len / 2;

                // 拆分前一半
                List<V> headValues = values.subList(0 , min);
                ListForkJoinExecution<V,R> a = new ListForkJoinExecution(headValues, function);
                a.setResult(resultQueue);
                a.fork();
                resultQueue.offer(a);

                // 拆分后一半
                List<V> endValues = values.subList(min + 1 , len);
                ListForkJoinExecution<V,R> b = new ListForkJoinExecution(endValues, function);
                b.setResult(resultQueue);
                b.fork();
                resultQueue.offer(b);

                // 本次任务处理一个
                R r = function.apply(values.get(min));
                if (r != null) {
                    return r;
                }
            } else if (len == 2){

                List<V> headValues = values.subList(0 , 1);
                ListForkJoinExecution<V,R> a = new ListForkJoinExecution(headValues, function);
                a.setResult(resultQueue);
                a.fork();
                resultQueue.offer(a);

                // 拆分后一半
                List<V> endValues = values.subList(1 , 2);
                ListForkJoinExecution<V,R> b = new ListForkJoinExecution(endValues, function);
                b.setResult(resultQueue);
                b.fork();
                resultQueue.offer(b);

            } else if(len == 1){

                R r = function.apply(values.get(0));
                if (r != null) {
                    return r;
                }
            }
        }catch (Exception e){
            log.error(e.getMessage(), e);
        }

        return null;
    }
}
```

2. 执行forkjoin任务

```java
import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.ForkJoinPool;

@Slf4j
public class ForkJoinPoolRun {

    /**
     * 并行处理列表方法
     * @param task  任务
     * @param <V>   参数对象类型
     * @param <R>   返回对象类型
     * @return
     */
    public static <V, R> List<R> run(ListForkJoinExecution<V, R> task){
        return run(8, task);
    }

    public static <V, R> List<R> run(int poolSize, ListForkJoinExecution<V, R> task){
        ForkJoinPool pool = new ForkJoinPool(poolSize);

        List<R> result = Lists.newArrayList();
        ConcurrentLinkedQueue<ListForkJoinExecution<V, R>> resultQueue = new ConcurrentLinkedQueue<>();
        try {

            task.setResult(resultQueue);
            // 执行
            R r = pool.submit(task).get();
            // 没有结算结果的不追加到结果集中
            if (r != null) {
                result.add(r);
            }

            while (resultQueue.iterator().hasNext()) {
                ListForkJoinExecution<V, R> poll = resultQueue.poll();
                if (poll != null) {
                    R join = poll.join();
                    // 没有结算结果的不追加到结果集中
                    if (join != null) {
                        result.add(join);
                    }
                }

            }

            pool.shutdown();

            return result;
        } catch (Exception e) {
            log.error("遍历处理任务异常！", e);
        }

        return result;
    }

    /**
     * 并执行无返回方法
     * @param task  任务
     * @param <R>   返回对象类型
     * @return
     */
    public static <R> void run(VoidForkJoinExecution<R> task){
        run(8, task);
    }

    public static <R> void run(int poolSize, VoidForkJoinExecution<R> task){
        ForkJoinPool pool = new ForkJoinPool(poolSize);

        try {
            // 执行
            pool.submit(task);

            while (!pool.isTerminated()){
                pool.shutdown();
            }
        } catch (Exception e) {
            log.error("遍历处理任务异常！", e);
        }
    }
}

```
