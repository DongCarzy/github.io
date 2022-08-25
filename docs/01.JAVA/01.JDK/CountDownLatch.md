---
title: CountDownLatch
date: 2022-08-24 22:29:57
permalink: /pages/ea5bd1/
categories:
  - JAVA
  - JDK
tags:
  -  源码阅读
---
# CountDownLatch

> jdk17

允许一个或者多个线程去等待其他线程完成操作.

## 构造函数

```java
   public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

`CountDownLatch`实例化时,其实是实例化了一个内部的 `Sync` 对象, 并将形参透传给`Sync`, `Sync` 其实是一个 `AQS` 子类

```JAVA
private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
          // 设置同步状态的值
            setState(count);
        }

        int getCount() {
            // 设置同步状态的值
            return getState();
        }

        // 加共享锁   尝试获取同步, 且状态值为0比欧式成功,否则失败. 
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        // 释放加共享锁  尝试释放同步状态,
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                // 状态为0, 直接放回
                if (c == 0)
                    return false;
                // 通过cas将同步状态值减1
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

## 重点方法 `countDown` `await`

`countDown` `await` 是 `CountDownLatch` 类中最重要的两个方法,一个是将同步状态值减1, 一个则是阻塞等待.

### countDown

内部实质上是调用 `Sync` 的 `tryReleaseShared(1)`, 将状态值减1

```Java
    /**  CountDownLatch */
   public void countDown() {
        // 调动 AQS 的 releaseShared 方法
        sync.releaseShared(1);
    }

    /**  AbstractQueuedSynchronizer */
    public final boolean releaseShared(int arg) {
      // 调用子类的 CountDownLatch.Sync 自旋减同步状态值
      if (tryReleaseShared(arg)) {
          signalNext(head);
          return true;
      }
      return false;
    }
```

### await

> 阻塞线程,等待状态值变为0

```java
  /**  CountDownLatch */
  public void await() throws InterruptedException {
      sync.acquireSharedInterruptibly(1);
  }

  /**  AbstractQueuedSynchronizer */
  public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
  
  // 中断了则直接抛出 InterruptedException 异常
  if (Thread.interrupted() ||

      // 尝试 `Sync` 类中重写的 `tryAcquireShared` 尝试加锁, 
      // 如果加锁失败, 调用 acquire
      // 加锁成功,直接放回,结束阻塞状态
      (tryAcquireShared(arg) < 0 &&

        // 阻塞当前线程
        acquire(null, arg, true, true, false, 0L) < 0))
      throw new InterruptedException();
  }
```

## 使用案例

1. 创建一个线程池,用于执行并发任务
2. 声明一个 `CountDownLatch`, 大小为任务的数量(10)
3. 往线程池中丢10个任务
4. 主线程通过 await 阻塞等待任务执行完毕
5. 关闭线程池,结束程序

```java
import java.time.LocalDateTime;
import java.util.concurrent.*;

public class Test {

    static final ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 10, 3,
            TimeUnit.SECONDS, new ArrayBlockingQueue<>(10), Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

    public static void main(String[] args) throws InterruptedException {
        final int size = 10;

        final CountDownLatch latch = new CountDownLatch(size);
        System.out.println("处理开始");

        for (int i = 0; i < size; i++) {
            executor.execute(() -> {
                try {
                    Thread.sleep(1000);
                    System.out.println(Thread.currentThread().getName() + "---" + LocalDateTime.now());
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } finally {
                    latch.countDown();
                }
            });
        };

        latch.await();
        System.out.println("处理完成");
        executor.shutdownNow();
    }
}
```

从输出的结果可以看出, 任务是并发执行的,  `处理完成` 这个业务会等待 `latch.await()`的阻塞状态结束

```
处理开始
pool-1-thread-7---2022-08-25T14:30:21.750001
pool-1-thread-5---2022-08-25T14:30:21.750052
pool-1-thread-3---2022-08-25T14:30:21.749940
pool-1-thread-10---2022-08-25T14:30:21.749927
pool-1-thread-9---2022-08-25T14:30:21.750023
pool-1-thread-4---2022-08-25T14:30:21.750008
pool-1-thread-8---2022-08-25T14:30:21.749924
pool-1-thread-1---2022-08-25T14:30:21.750070
pool-1-thread-6---2022-08-25T14:30:21.750029
pool-1-thread-2---2022-08-25T14:30:21.749884
处理完成
```
