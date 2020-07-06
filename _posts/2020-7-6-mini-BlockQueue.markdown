---
layout:     post
title:      "mini BlockQueue"
subtitle:   " \"mini BlockQueue\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - BlockQueue
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)


# 简易版 BlockQueue

```jsx

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Create with IDEA
 *
 * @author linyi
 * @date 2020-05-26
 **/
public class MiniBlockQueue implements BlockQueue {

    // 重入锁
    private Lock lock = new ReentrantLock();

    // 生产者: 当生产者发现队列数据,满了之后会挂起,等待消费者消费完之后,唤醒,继续添加数据
    private Condition notFull = lock.newCondition();

    // 消费者: 当消费者发现队列没数据时,会挂起,等待生产者生产新数据之后,唤醒消费者线程
    private Condition notEmpty = lock.newCondition();

    // 消息数组
    private Object[] queues;

    // 消息可存数量
    private int size;

    // 可消费的总数
    // 下一次数据存放的位置
    // 下一次消费数据的位置
    private int count,putPtr,takePtr;

    @Override
    public void put(Object element) throws InterruptedException {
        lock.lock();
        try {
            // 队列满了,等待消费完毕,唤醒
            if (size == count){
                notFull.await();
            }
            // 队列没满,放入队列
            queues[putPtr] = element;
            // 放入元素 +1
            putPtr ++;

            // 元素放满了,且前面的都被消费了
            if (putPtr == size) putPtr = 0;

            // 总元素
            count ++;

            //唤醒消费者,可以消费数据了
            notEmpty.signal();
        }finally {
            lock.unlock();
        }
    }

    @Override
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            if (count == 0){
                notEmpty.await();
            }
            // 消费元素
            Object element = queues[takePtr];

            //消费位置+1
            takePtr++;

            //如果到了队尾,重新指向头
            if (takePtr == size) takePtr = 0;

            // 可消费数量-1
            count--;

            //唤醒生产者,可以加数据源了
            notFull.signal();
            return element;
        }finally {
            lock.unlock();
        }
    }

    public MiniBlockQueue(Integer size) {
        this.size = size;
        this.queues = new Object [size];
    }

    public static void main(String[] args) {
        MiniBlockQueue queue = new MiniBlockQueue(10);

        Thread producer = new Thread(() -> {
            int i = 0;
            while(true) {
                i ++;
                if(i == 10) i = 0;
                try {
                    System.out.println("生产数据：" + i);
                    queue.put(i);
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                }
            }
        });
        producer.start();

        Thread consumer = new Thread(() -> {
            while(true) {
                try {
                    Object result = queue.take();
                    System.out.println("消费者消费：" + result);
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        consumer.start();
    }
}
```

```java
package com.xhxf.utils;

/**
 * Create with IDEA
 *
 * @author linyi
 * @date 2020-05-26
 **/
public interface BlockQueue<T> {

    void put(T element) throws InterruptedException;

    T take() throws InterruptedException;
}
```