---
layout:     post
title:      "CountDownLatch"
subtitle:   " \"CountDownLatch\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - CountDownLatch
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# CountDownLatch

内部使用sync 实现了 AQS

```java

// 构造设置栅栏		
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
	    }

private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        // 设置栅栏
        Sync(int count) {
            setState(count);
        }
				
        // 获取剩余栅栏
        int getCount() {
            return getState();
        }
				
        // 判断栅栏是否还在  还在返回 -1 不在返回 1
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
				
        // 判断当前是不是栅栏前最后一个线程
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

// await
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

// AQS的方法
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        // 被中断抛出异常
        if (Thread.interrupted())
            throw new InterruptedException();
        // 判断栅栏是否还在  
        // true state < 0  -1 还在 挂起,等待唤醒
        // false state = 0 1 栅栏消除,直接通过  
        if (tryAcquireShared(arg) < 0)
            // 入队挂起
            doAcquireSharedInterruptibly(arg);
    }

// 入队挂起
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 入队  ReentrantLock 讲过了
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                // 入队的前置节点
                final Node p = node.predecessor();
                // 前置是头结点
                if (p == head) {
                    // 判断栅栏
                    int r = tryAcquireShared(arg);
										// r 为 1 
                    if (r >= 0) {
                        // 设置自己为头结点并向下传播 ,唤醒队列中的node结点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

// 设置自己为头结点并向下传播
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
     // 设置自己为头结点
        setHead(node);
       // propagate = 1
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            // 当前节点的后继结点
            Node s = node.next;
            // true 最后一个节点
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

private void doReleaseShared() {
       // 唤醒park的节点
        for (;;) {
            // 头结点引用
            Node h = head;
            // 条件1 : true 阻塞队列不为空 false 创建出来后,未调用await() 方法
            // 条件2: 除了头结点之外,还有其他节点
            if (h != null && h != tail) {
                // 头结点状态
                int ws = h.waitStatus;
                // 状态为唤醒后继结点
                if (ws == Node.SIGNAL) {
                    // cas 修改节点状态为 默认  ,可能失败,存在竞争
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    // 唤醒后继结点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            //条件成立：
            //1.说明刚刚唤醒的 后继节点，还没执行到 setHeadAndPropagate方法里面的 设置当前唤醒节点为head的逻辑。
            //这个时候，当前线程 直接跳出去...结束了..
            //此时用不用担心，唤醒逻辑 在这里断掉呢？、
            //不需要担心，因为被唤醒的线程 早晚会执行到doReleaseShared方法。

            //2.h == null  latch创建出来后，没有任何线程调用过 await() 方法之前，
            //有线程调用latch.countDown()操作 且触发了 唤醒阻塞节点的逻辑..
            //3.h == tail  -> head 和 tail 指向的是同一个node对象

            //条件不成立：
            //被唤醒的节点 非常积极，直接将自己设置为了新的head，此时 唤醒它的节点（前驱），执行h == head 条件会不成立..
            //此时 head节点的前驱，不会跳出 doReleaseShared 方法，会继续唤醒 新head 节点的后继...
            if (h == head)                   // loop if head changed
                break;
        }
    }
}
```
