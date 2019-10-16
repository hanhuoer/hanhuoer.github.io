---
layout: post
title: Java - 多线程编程 - CountDownLatch
categories: [Java]
category: Java
tags: [Java]
keywords: Java,多线程,多线程编程
---

## CountDownLatch 是什么？

> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

提供一种同步辅助功能，它允许一个或多个线程等待，直到其它线程完成一组操作之后。

## CountDownLatch 如何使用？

其实在 `java.util.concurrent.CountDownLatch` 类注释中已经给了一些典型的用例。

下面有一个需求：**主线程中启动一些子线程，接下来主线程会等待所有子线程工作完毕之后继续运行。**

测试使用的代码

```java

import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.stream.IntStream;

/**
 * @author H
 */
public class CountDownLatchTest {

    private static final Random RANDOM = new Random(System.currentTimeMillis());

    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(5);

        System.out.println("[main] Prepare handle multi-thread tasks.");

        IntStream.rangeClosed(1, 5)
                .forEach(i ->
                        new Thread(() -> {
                            try {
                                System.out.println("[task-start] " + Thread.currentThread().getName() + " is working.");
                                Thread.sleep(RANDOM.nextInt(1000));
                                System.out.println("[task-finished] " + Thread.currentThread().getName() + " is finished.");
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            countDownLatch.countDown();
                        }, String.valueOf(i)).start()
                );

        countDownLatch.await();
        System.out.println("[main] Finished.");
    }

}

// [main] Prepare handle multi-thread tasks.
// [task-start] 2 is working.
// [task-start] 1 is working.
// [task-start] 3 is working.
// [task-start] 4 is working.
// [task-start] 5 is working.
// [task-finished] 1 is finished.
// [task-finished] 5 is finished.
// [task-finished] 2 is finished.
// [task-finished] 3 is finished.
// [task-finished] 4 is finished.
// [main] Finished.

```

## CountDownLatch 的原理是什么？

在 JDK 源码中，`CountDownLatch` 是通过计数器实现的。 线程会在`await()` 方法调用处阻塞；每当调用一次 `countDown()` 方法，那么将会触发一次释放操作，计数器也就会减一，直到由于调用 `countDown()` 方法而导致计数器达到零为止，之后所有等待的线程都将会被释放。

需要注意，在获取 `CountDownLatch` 实例的时候需要在构造方法中传入线程的数量。这是一个固定的值并且不能为负数，`CountDownLatch` 没有提供重置此数量的方法，如果有需要重置操作可以去看看 `java.util.concurrent.CyclicBarrier`


`CountDownLatch` 的构造方法

```java
/**
 * Constructs a {@code CountDownLatch} initialized with the given count.
 *
 * @param count the number of times {@link #countDown} must be invoked
 *        before threads can pass through {@link #await}
 * @throws IllegalArgumentException if {@code count} is negative
 */
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```


## 如何自己实现一个 CountDownLatch ？

下面给出一个简易版的 `CountDownLatch`，每次调用 `down()` 方法，计数器自加操作，`await()` 方法调用出一直阻塞，直到计数器等于线程总数才会获得释放。


```java

public class CustomCountDownLatch {
    private final int total;

    private int counter = 0;

    public CustomCountDownLatch(int total) {
        this.total = total;
    }

    public void down() {
        synchronized (this) {
            counter++;
            notifyAll();
        }
    }

    public void await() throws InterruptedException {
        synchronized (this) {
            while (counter != total) {
                wait();
            }
        }
    }
}

```

测试用例

```java
import java.util.Random;
import java.util.stream.IntStream;

/**
 * @author H
 */
public class CustomCountDownLatchTest {

    private static final Random RANDOM = new Random(System.currentTimeMillis());

    public static void main(String[] args) throws InterruptedException {
        final CustomCountDownLatch customCountDownLatch = new CustomCountDownLatch(5);

        System.out.println("[main] Prepare handle multi-thread tasks.");

        IntStream.rangeClosed(1, 5)
                .forEach(i ->
                        new Thread(() -> {
                            try {
                                System.out.println("[task-start] " + Thread.currentThread().getName() + " is working.");
                                Thread.sleep(RANDOM.nextInt(1000));
                                System.out.println("[task-finished] " + Thread.currentThread().getName() + " is finished.");
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            customCountDownLatch.down();
                        }, String.valueOf(i)).start()
                );

        customCountDownLatch.await();
        System.out.println("[main] Finished.");
    }
}

// [main] Prepare handle multi-thread tasks.
// [task-start] 2 is working.
// [task-start] 3 is working.
// [task-start] 4 is working.
// [task-start] 5 is working.
// [task-start] 1 is working.
// [task-finished] 4 is finished.
// [task-finished] 2 is finished.
// [task-finished] 3 is finished.
// [task-finished] 1 is finished.
// [task-finished] 5 is finished.
// [main] Finished.

```
