---
title: Java实现乐观互斥Key锁
date: 2018-07-14 15:42:02
categories: 
- 后端
tags:
- Java
---

# 简介

java中的几种锁：synchronized，ReentrantLock，ReentrantReadWriteLock已基本可以满足编程需求，但其粒度都太大，同一时刻只有一个线程能进入同步块，加锁后性能受到太大的影响。这对于某些高并发的场景并不适用。本文实现了一个基于KEY（主键）的互斥锁，具有更细的粒度，在缓存或其他基于KEY的场景中有很大的用处。下面将讲解这个锁的设计和实现

# 分段锁

```java
/**
 * Created by hhbbz on 2018/7/13.
 * @Explain: key锁（要保证key的hashCode不变,否则无法释放锁。即加锁之后不要手动更改lockMap）
 */
@Component
public class LoadKeyLock<T> {
    //默认分段数量
    private Integer segments = 16;
    private final HashMap<Integer, ReentrantLock> lockMap = new HashMap<>();
    public LoadKeyLock() {
        init(null, false);
    }
    public LoadKeyLock(Integer counts, boolean fair) {
        init(counts, fair);
    }
    private void init(Integer counts, boolean fair) {
        if (counts != null) {
            segments = counts;
        }
        for (int i = 0; i < segments; i++) {
            lockMap.put(i, new ReentrantLock(fair));
        }
    }
    public void lock(T key) {
        ReentrantLock lock = lockMap.get(key.hashCode() % segments);
        lock.lock();
    }
    public void unlock(T key) {
        ReentrantLock lock = lockMap.get(key.hashCode() % segments);
        lock.unlock();
    }
}
```

# 哈希锁

上述分段锁的基础上发展起来的第二种锁策略，目的是实现真正意义上的细粒度锁。每个哈希值不同的对象都能获得自己独立的锁。在测试中，在被锁住的代码执行速度飞快的情况下，效率比分段锁慢 30% 左右。如果有长耗时操作，感觉表现应该会更好。代码如下：

```java
public class HashLock<T> {
    private boolean isFair = false;
    private final SegmentLock<T> segmentLock = new SegmentLock<>();//分段锁
    private final ConcurrentHashMap<T, LockInfo> lockMap = new ConcurrentHashMap<>();

    public HashLock() {
    }

    public HashLock(boolean fair) {
        isFair = fair;
    }

    public void lock(T key) {
        LockInfo lockInfo;
        segmentLock.lock(key);
        try {
            lockInfo = lockMap.get(key);
            if (lockInfo == null) {
                lockInfo = new LockInfo(isFair);
                lockMap.put(key, lockInfo);
            } else {
                lockInfo.count.incrementAndGet();
            }
        } finally {
            segmentLock.unlock(key);
        }
        lockInfo.lock.lock();
    }

    public void unlock(T key) {
        LockInfo lockInfo = lockMap.get(key);
        if (lockInfo.count.get() == 1) {
            segmentLock.lock(key);
            try {
                if (lockInfo.count.get() == 1) {
                    lockMap.remove(key);
                }
            } finally {
                segmentLock.unlock(key);
            }
        }
        lockInfo.count.decrementAndGet();
        lockInfo.unlock();
    }

    private static class LockInfo {
        public ReentrantLock lock;
        public AtomicInteger count = new AtomicInteger(1);

        private LockInfo(boolean fair) {
            this.lock = new ReentrantLock(fair);
        }

        public void lock() {
            this.lock.lock();
        }

        public void unlock() {
            this.lock.unlock();
        }
    }
}
```

# 弱引用锁

哈希锁因为引入的分段锁来保证锁创建和销毁的同步，总感觉有点瑕疵，所以写了第三个锁来寻求更好的性能和更细粒度的锁。这个锁的思想是借助java的弱引用来创建锁，把锁的销毁交给jvm的垃圾回收，来避免额外的消耗。

有点遗憾的是因为使用了ConcurrentHashMap作为锁的容器，所以没能真正意义上的摆脱分段锁。这个锁的性能比 HashLock 快10% 左右。锁代码：

```java
/**
 * 弱引用锁，为每个独立的哈希值提供独立的锁功能
 */
public class WeakHashLock<T> {
    private ConcurrentHashMap<T, WeakLockRef<T, ReentrantLock>> lockMap = new ConcurrentHashMap<>();
    private ReferenceQueue<ReentrantLock> queue = new ReferenceQueue<>();

    public ReentrantLock get(T key) {
        if (lockMap.size() > 1000) {
            clearEmptyRef();
        }
        WeakReference<ReentrantLock> lockRef = lockMap.get(key);
        ReentrantLock lock = (lockRef == null ? null : lockRef.get());
        while (lock == null) {
            lockMap.putIfAbsent(key, new WeakLockRef<>(new ReentrantLock(), queue, key));
            lockRef = lockMap.get(key);
            lock = (lockRef == null ? null : lockRef.get());
            if (lock != null) {
                return lock;
            }
            clearEmptyRef();
        }
        return lock;
    }

    @SuppressWarnings("unchecked")
    private void clearEmptyRef() {
        Reference<? extends ReentrantLock> ref;
        while ((ref = queue.poll()) != null) {
            WeakLockRef<T, ? extends ReentrantLock> weakLockRef = (WeakLockRef<T, ? extends ReentrantLock>) ref;
            lockMap.remove(weakLockRef.key);
        }
    }

    private static final class WeakLockRef<T, K> extends WeakReference<K> {
        final T key;

        private WeakLockRef(K referent, ReferenceQueue<? super K> q, T key) {
            super(referent, q);
            this.key = key;
        }
    }
}
```

# 适合耗时长场景的互斥key锁

一个细粒度的锁，在某些场景能比synchronized，ReentrantLock等获得更高的并行度更好的性能

```java
public class KeyLock<K> {
    // 保存所有锁定的KEY及其信号量
    private final ConcurrentMap<K, Semaphore> map = new ConcurrentHashMap<K, Semaphore>();
    // 保存每个线程锁定的KEY及其锁定计数
    private final ThreadLocal<Map<K, LockInfo>> local = new ThreadLocal<Map<K, LockInfo>>() {
        @Override
        protected Map<K, LockInfo> initialValue() {
            return new HashMap<K, LockInfo>();
        }
    };

    /**
     * 锁定key，其他等待此key的线程将进入等待，直到调用{@link #unlock(K)}
     * 使用hashcode和equals来判断key是否相同，因此key必须实现{@link #hashCode()}和
     * {@link #equals(Object)}方法
     * 
     * @param key
     */
    public void lock(K key) {
        if (key == null)
            return;
        LockInfo info = local.get().get(key);
        if (info == null) {
            Semaphore current = new Semaphore(1);
            current.acquireUninterruptibly();
            Semaphore previous = map.put(key, current);
            if (previous != null)
                previous.acquireUninterruptibly();
            local.get().put(key, new LockInfo(current));
        } else {
            info.lockCount++;
        }
    }
    /**
     * 释放key，唤醒其他等待此key的线程
     * @param key
     */
    public void unlock(K key) {
        if (key == null)
            return;
        LockInfo info = local.get().get(key);
        if (info != null && --info.lockCount == 0) {
            info.current.release();
            map.remove(key, info.current);
            local.get().remove(key);
        }
    }
 
    /**
     * 锁定多个key
     * 建议在调用此方法前先对keys进行排序，使用相同的锁定顺序，防止死锁发生
     * @param keys
     */
    public void lock(K[] keys) {
        if (keys == null)
            return;
        for (K key : keys) {
            lock(key);
        }
    }

    /**
     * 释放多个key
     * @param keys
     */
    public void unlock(K[] keys) {
        if (keys == null)
            return;
        for (K key : keys) {
            unlock(key);
        }
    }

    private static class LockInfo {
        private final Semaphore current;
        private int lockCount;

        private LockInfo(Semaphore current) {
            this.current = current;
            this.lockCount = 1;
        }
    }
}
```