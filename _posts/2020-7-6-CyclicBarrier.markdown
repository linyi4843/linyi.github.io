---
layout:     post
title:      "CyclicBarrier"
subtitle:   " \"CyclicBarrier\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - CyclicBarrier
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# CyclicBarrier

```java
		// 代 是否破掉
		private static class Generation {
        boolean broken = false;
    }

    /** The lock for guarding barrier entry */
		// 实现是Condition 条件队列
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    /** The number of parties */
		// 代 阈值
    private final int parties;
    /* The command to run when tripped */
		// 传入的 runnable 对象
    private final Runnable barrierCommand;
    /** The current generation */
		// 创建 代
    private Generation generation = new Generation();
		
		// 还剩多少到 0 , count = parties --
		private int count;

// 主要实现方法
// 是否启动超时机制,超时时间
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
				// Condition 队列,必须要先获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
						// 获取代对象
            final Generation g = generation;
						// 代 已经破掉,抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
						// 被中断也抛出异常
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
						// --
            int index = --count;
						// 最后一个线程,要干脏活,唤醒其余线程,并重置 代
            if (index == 0) {  // tripped
								// 是否抛出异常  : 自传入的对象,运行可能出现异常
                boolean ranAction = false;
                try {
										// 自传对象
                    final Runnable command = barrierCommand;
										// (可以为空)
                    if (command != null)
												// 运行自传线程逻辑
                        command.run();
										// 未抛出异常
                    ranAction = true;
										// 唤醒所有线程,并重置代
                    nextGeneration();
                    return 0;
                } finally {
										
                    if (!ranAction)
												// 发生异常了
                        breakBarrier();
                }
            }
						
						// 到这里说明不是最后一个线程
            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
										// 超时机制
										// fasle 挂起
                    if (!timed)
                        trip.await();
										// 是否超时
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
										// 进来就是抛出了中断异常
										// 代 是否发生变化
										// 代 是否被打破
                    if (g == generation && ! g.broken) {
												// 
                        breakBarrier();
                        throw ie;
                    } else {
                        // 1 代 发生变化
												// 2 代 被打破
												// 设置中断标记
                        Thread.currentThread().interrupt();
                    }
                }
								// 被打破
                if (g.broken)
                    throw new BrokenBarrierException();
								// 代 发生改变
                if (g != generation)
                    return index;
								// 超时了,移到阻塞队列,等待锁唤醒
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

// 唤醒所有线程,并重置代
private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

// 唤醒所有线程
public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }

// 
private void breakBarrier() {
				// 设置代 被打破
        generation.broken = true;
				 //重置
        count = parties;
				// 唤醒所有线程
        trip.signalAll();
    }
```