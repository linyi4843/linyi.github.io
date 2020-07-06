---
layout:     post
title:      "mini ReentrantLock"
subtitle:   " \"mini ReentrantLock\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - ReentrantLock
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# 简易版ReentrantLock

```java
package learn.lock;

import sun.misc.Unsafe;

import java.lang.reflect.Field;
import java.util.concurrent.locks.LockSupport;

/**
 * Created with IDEA
 *
 * @author: linyi
 * @Email: linyi4843@gmail.com
 */
public class MiniReentrantLockTest implements Lock{

    // 线程抢占状态  0 无锁状态  > 0 有锁状态
    private volatile int state;

    // 当前持有锁线程
    private Thread exclusiveOwnerThread;

    //等待队列为双向链表
    //头结点
    private Node head;
    private Node tail;

    // node节点
    static final class Node {
        //前节点
        Node prev;
        //后节点
        Node next;
        //当前线程
        Thread thread;

        public Node(Thread thread) {
            this.thread = thread;
        }

        public Node() {
        }
    }

    /**
     * 获取锁
     */
    @Override
    public void lock() {
        acquire(1);
    }

    // 获取锁实际实现
    private void acquire(int arg) {
        // 尝试获取锁
        // ! false获取锁失败
        // ! true获取锁成功
        if(!tryAcquire(arg)) {
            // 获取锁失败,将当前线程加入队列
            Node node = addWaiter();
            // 入队成功,开始排队
            acquireQueued(node, arg);
        }
    }

    //尝试获取锁
    private boolean tryAcquire(int arg) {
        // 如果当前无竞争处于无锁状态
        if(state == 0) {

            // 判断当前时间,有无其他线程进行了加锁 -- cas获取锁
            // 当前线程是否是head的下一个线程
            if(!hasQueuedPredecessor() && compareAndSetState(0, arg)) {
                // 把当前线程设为占有锁线程
                this.exclusiveOwnerThread = Thread.currentThread();
                return true;
            }
        // 前提条件,当前线程已经持有锁
        } else if(Thread.currentThread() == this.exclusiveOwnerThread) {
            //累加锁标识state
            int c = getState();
            c = c + arg;
            this.state = c;
            return true;
        }
        // 当前存在竞争
        return false;
    }

    private void acquireQueued(Node node, int arg) {
        // 自旋获取锁
        for(;;) {
            // 当前节点的上一个节点
            Node pred = node.prev;
            // 当前节点的上一个节点是head
            // 尝试获取锁成功
            if(pred == head && tryAcquire(arg)) {
                // 设置头结点
                setHead(node);
                // 上一个持有锁的node,置空
                pred.next = null; // help GC
                return;
            }
            // 当前节点不是head的下一个节点,
            System.out.println("线程：" + Thread.currentThread().getName() + "，挂起！");
            // 挂起当前线程,等待释放线程锁之后,唤醒该线程
            LockSupport.park();
            System.out.println("线程：" + Thread.currentThread().getName() + "，唤醒！");
            // 释放锁之后自旋继续获取锁
        }
    }

    private Node addWaiter() {
        // 当前线程加入队列
        Node newNode = new Node(Thread.currentThread());
        // 尾节点引用,后续将新节点设为尾节点
        Node pred = tail;
        // 存在队列
        if(pred != null) {
            // 设为新节点的前一个节点
            newNode.prev = pred;
            // cas 设置尾节点
            if(compareAndSetTail(pred, newNode)) {
                // 设置前节点的下一个节点
                pred.next = newNode;
                return newNode;
            }
        }
        // 尾节点为空,或者cas设置节点失败
        enq(newNode);
        return newNode;
    }

    private void enq(Node node) {
        //自旋加入队列
        for(;;) {
            // 尾节点为空
            if(tail == null) {
                // 头结点设为尾节点
                if(compareAndSetHead(new Node())) {
                    tail = head;
                }
            } else {
                // 参照addWaiter
                Node pred = tail;
                if(pred != null) {
                    node.prev = pred;

                    if(compareAndSetTail(pred, node)) {
                        pred.next = node;
                        return;
                    }
                }
            }
        }
    }

    // 判断是否已经有排队线程
    private boolean hasQueuedPredecessor() {
        // 头结点
        Node h = head;
        // 尾节点
        Node t = tail;
        // 头结点的下一个节点
        Node s;
        // h != t
        // true 已经存在排队线程
        // false 无排队线程  h == t == null / h == t == head
        // (s = h.next) == null || s.thread != Thread.currentThread()
        // (s = h.next) == null --> 第一个排队线程初始化了持锁线程,但还未把自己加入到队列
        // s.thread != Thread.currentThread()  --> 当前线程是head的next,可以竞争锁
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
    }

    // 释放锁
    @Override
    public void unlock() {
        release(1);
    }

    //释放锁逻辑
    private void release(int arg) {
        // 尝试释放锁
        if(tryRelease(arg)) {
            // 近来已完全释放锁
            Node head = this.head;
            // head的下一个元素不为空
            if(head.next != null) {
                // 释放被挂起的线程,竞争锁
                unparkSuccessor(head);
            }
        }
    }

    // 释放被挂起线程
    private void unparkSuccessor(Node node) {
        // 当前加锁线程节点的下一个节点
        Node s = node.next;
        // 不为空 且 线程也不为空
        if(s != null && s.thread != null) {
            // 释放挂起的线程
            LockSupport.unpark(s.thread);
        }
    }

    // 尝试释放锁
    private boolean tryRelease(int arg) {
        // state表示
        int c = getState() - arg;

        // 当前线程是不是加锁线程
        if(getExclusiveOwnerThread() != Thread.currentThread()) {
            // 如果不是不释放
            throw new RuntimeException("fuck you! must getLock!");
        }
        // 等于0, 完全释放锁
        if(c == 0) {
            // 持有锁线程置空
            this.exclusiveOwnerThread = null;
            // 标识设为0
            this.state = c;
            return true;
        }
        // 为完全释放,等待继续释放
        this.state = c;
        return false;
    }

    // 设置头结点
    private void setHead(Node node) {
        // 当前节点设为头结点(第一个线程或者head的next节点)
        this.head = node;
        // 因为当前线程已经持有锁了,所以可设置为null
        node.thread = null;
        // 头结点没有前置节点
        node.prev = null;
    }

    // 获取当前锁状态
    public int getState() {
        return state;
    }

    // 谁知持有锁线程
    public Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }

    public Node getHead() {
        return head;
    }

    public Node getTail() {
        return tail;
    }

    // cas 操作
    private static final Unsafe unsafe;
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);

            stateOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("tail"));

        } catch (Exception ex) { throw new Error(ex); }
    }

    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}
```