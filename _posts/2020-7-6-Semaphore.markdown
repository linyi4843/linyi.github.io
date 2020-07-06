---
layout:     post
title:      "Semaphore"
subtitle:   " \"Semaphore\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - Semaphore
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# Semaphore

主要实现 sync 继承自 AQS

```java
// 构造方法 默认非公平模式   permits 设置阈值 使用AQS的setState()
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

// 可选参数 true 公平  false 非公平
public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

// 栅栏
public void acquire() throws InterruptedException {
				// AQS 
        sync.acquireSharedInterruptibly(1);
    }

// AQS 共享模式
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
				// 
        if (tryAcquireShared(arg) < 0)
						// 挂起
            doAcquireSharedInterruptibly(arg);
    }

// 
protected int tryAcquireShared(int acquires) {
            for (;;) {
								// 队列有人直接返回,这是公平模式,不能插队
                if (hasQueuedPredecessors())
                    return -1;
								// 获取信号量剩余数量
                int available = getState();
								// 减去当前所需之后,是否够用
                int remaining = available - acquires;
								// 负数 不够用,返回挂起
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
				 // 入队
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
								// 前置节点获取
                final Node p = node.predecessor();
								// 头结点
                if (p == head) {
										// 再次判断信号量是否够用
                    int r = tryAcquireShared(arg);
										// 够用
                    if (r >= 0) {
												// 设置自己为头结点,并唤醒后后继结点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
								// 找一个好爹然后挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
										// 挂起
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
						// 响应中断出队
            if (failed)
                cancelAcquire(node);
        }
    }

// 释放持有的信号量
public void release() {
        sync.releaseShared(1);
    }

public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
//
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
								// 当前信号量个数
                int current = getState();
								// 越界了
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
								// 加入释放的信号量
                if (compareAndSetState(current, next))
                    return true;
            }
        }

// 唤醒后续结点
private void doReleaseShared() {
  
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```