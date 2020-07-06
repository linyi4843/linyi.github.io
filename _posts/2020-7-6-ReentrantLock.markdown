---
layout:     post
title:      "ReentrantLock"
subtitle:   " \"ReentrantLock\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - ReentrantLock
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# ReentrantLock

重入锁- 独占模式

刀哥李  

Node属性 

```java
		static final class Node {  
                // 共享模式
                static final Node SHARED = new Node();
                // 独占模式
                static final Node EXCLUSIVE = null;
        
                //当前节点位于取消状态
                static final int CANCELLED =  1;
                // 表示当前节点需要唤醒他的后继节点( SIGNAL 表示后继节点的状态,需要将其唤醒) 
                static final int SIGNAL    = -1;
                /** waitStatus value to indicate thread is waiting on condition */
                static final int CONDITION = -2;
                
                static final int PROPAGATE = -3;
        
                // 状态 
                // 默认状态 0
                // 唤醒后继节点 -1(SIGNAL)
                // 取消状态 1 cancelled 
                volatile int waitStatus;
        
                // 因为node需要组建成 fifo 对列,所以 prev 指向前置节点 
                volatile Node prev;
        
                // 因为node需要组建成 fifo 对列,所以 next 指向后置节点
                volatile Node next;
         
               // 当前node封装的线程
                volatile Thread thread;
        
                // -- 未用到
                Node nextWaiter;
        
               
                final boolean isShared() {
                    return nextWaiter == SHARED;
                }
        
                final Node predecessor() throws NullPointerException {
                    Node p = prev;
                    if (p == null)
                        throw new NullPointerException();
                    else
                        return p;
                }
        
                Node() {    // Used to establish initial head or SHARED marker
                }
        
                Node(Thread thread, Node mode) {     // Used by addWaiter
                    this.nextWaiter = mode;
                    this.thread = thread;
                }
        
                Node(Thread thread, int waitStatus) { // Used by Condition
                    this.waitStatus = waitStatus;
                    this.thread = thread;
                }
            }
```

公共属性

```java
		// 头结点,持锁线程
		private transient volatile Node head;

        // 尾节点
        private transient volatile Node tail;

         // 资源
		// 独占模式 0 表示未加锁状态, > 0 表示已经加锁
        private volatile int state;

		// 当前持锁线程
		private transient Thread exclusiveOwnerThread;
```

// 公平模式

```java
// 公平锁
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

				public final void acquire(int arg) {
                // 条件一: true 成功获取锁, fasle 获得锁失败
                // 条件二: addwaiter 将线程封装node入队
                //					acquireQueued  挂起当前线程,唤醒后续线程
                //       返回true 则表示挂起过程中线程被中断过,fasle,未被中断过
		        if (!tryAcquire(arg) &&
		            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		            selfInterrupt();
		    }

				// 获取锁
        protected final boolean tryAcquire(int acquires) {
            // 当前线程
            final Thread current = Thread.currentThread();
            // 线程状态 0 无锁 > 0 有锁
            int c = getState();
            // 无锁
            if (c == 0) {
                // 判断当前是否需要入队等待
                // 1: true 表示存在等待着,入队等待
                // 2: false 无等待者,直接尝试获取锁
                if (!hasQueuedPredecessors() &&
                    // 抢占锁 true,fasle
                    compareAndSetState(0, acquires)) {
                    // 设置当前线程为独占者
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 条件  有锁 
            // 当前线程是不是独占线程, ReentrantLock 是可以重入的
            else if (current == getExclusiveOwnerThread()) {
                // 锁重量+1
                int nextc = c + acquires;
                // 越界判断,大于整数最大值时会变成负数
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                // 设置状态值
                setState(nextc);
                return true;
}
            // 1 加锁失败
            // 2 当前线程不是独占锁线程
            return false;
        }
    }

	

	// 入队
	private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 简单入队,当前队列不为空,直接加入对位
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
						// cas加锁
            if (compareAndSetTail(pred, node)) { 
                pred.next = node;
                return node;
            }
        }
        // 当前队列为空 or 加锁失败
        // 完整入队
        enq(node);
        return node;
    }

		// 完整入队
		private Node enq(final Node node) {

        // 自旋  第二个抢占锁线程给第一个线程擦屁股
        for (;;) {
            // 初始化队列
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 初始化完成,普通入队
                node.prev = t;
                // cas加锁
                if (compareAndSetTail(t, node)) {
                    // 入队最后
                    t.next = node;
                    // 返回前置节点
                    return t;
                }
            }
        }
    }

  // 
  final boolean acquireQueued(final Node node, int arg) {
        // true 表示抢占锁成功
        // false 失败 表示需要执行出队逻辑
        boolean failed = true;
        try {
            // 是否被中断
            boolean interrupted = false;
            for (;;) {
                // 得到前置节点
                final Node p = node.predecessor();
                // 表示当前节点是head的next节点
                // next节点有权获取锁
                // true 获取锁成功
                // false 说明未成功,仍需要被park
                if (p == head && tryAcquire(arg)) {
                    // 抢占成功设置自己为head节点
                    setHead(node);
                    p.next = null; // help GC
                    // 获取锁过程中没有异常 fasle
                    failed = false;
                    return interrupted;
                }
                // 1  当前线程是否需要挂起,true需要挂起,fasle不需要挂起
                // 2  挂起当前线程,返回中断标记
                // 正常唤醒,unpark   其他线程给当前线程一个中断信号
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // 唤醒后要执行上边逻辑删除前置节点,设置字节为头结点
                    
                    // 表示是被中断信号唤醒的
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
		
		// 判断需要是否需要被挂起
		private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 获取当前状态
        int ws = pred.waitStatus;
        // 需要唤醒后继节点,然后后面被 (parkAndCheckInterrupt )park掉,等待唤醒
        if (ws == Node.SIGNAL)
            
            return true;
        // 取消状态
        if (ws > 0) {
           //  找爸爸,获取到对应的signal节点,并将中间的除对处理
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
						// 当前节点为前节点的next
            pred.next = node;
        } else {
           // 如果当前状态为0,就会被设为 SIGNAL,继而下一次来执行上面的逻辑 
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

  //挂起当前线程,并返回中断信号
	private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

解锁 unlock()

```java
//解锁 sync集成aqs  aqs的解锁
public void unlock() {
        sync.release(1);
    }

// 解锁
public final boolean release(int arg) {
        // 尝试解锁
        if (tryRelease(arg)) {
            Node h = head;
            // 条件一 进来已经是最后一把锁了,头为空,投的状态不是默认状态
            // 条件二  不成立时,说明后面一定有后继节点
            if (h != null && h.waitStatus != 0)
								// 唤醒后继线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

// 尝试解锁
protected final boolean tryRelease(int releases) {
						// 是不是最后一个锁(可冲入的)
            int c = getState() - releases;
						// 判断当前线程是不是,当前独占线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
								// 抛出异常 不要搞事情好吧
                throw new IllegalMonitorStateException();
						// 返回是否还有加锁,返回标识
            boolean free = false;
            if (c == 0) {
                free = true;
								// 是最后一把锁 -> 设置独占线程为空
                setExclusiveOwnerThread(null);
            }
						// 设置剩余加锁数量,自身已经解锁成功
            setState(c);
            return free;
        }

// 唤醒后继节点
private void unparkSuccessor(Node node) {
        // 头节点的状态
        int ws = node.waitStatus;
        // SIGNAL 需要唤醒后继节点
        if (ws < 0)
            // 修改状态为默认状态0
            compareAndSetWaitStatus(node, ws, 0);

        // 后继节点
        Node s = node.next;
        // 节点空,1 可能是最后一个节点 2 新的节点正在入队,但还没有入队
        // 条件二 : 当前节点状态为取消状态 ,需要找一个合适的节点唤醒
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 从尾部遍历前置节点,寻找最近一个需要唤醒的节点
            // 可能找不到
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
            // 找到唤醒,找不到啥也不干
        }
        // 不为空直接唤醒
        if (s != null)
            LockSupport.unpark(s.thread);
    }

```

响应中断的锁

```java
		public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

		// 具体逻辑
		public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        // 被中断 抛出异常
        if (Thread.interrupted())
            throw new InterruptedException();
        // 尝试获取锁
        if (!tryAcquire(arg))
            // 获取到
            doAcquireInterruptibly(arg);
    }

		private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        // 入队
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
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
	
	// 响应中断  出队
	private void cancelAcquire(Node node) {
        // 不存在直接返回
        if (node == null)
            return;
        // 当前node线程设置为空
        node.thread = null;

      // 获取取消排队节点 的前置节点
        Node pred = node.prev;
        // 获得一个正常的前置节点,防止前置节点也被中断
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // 正常的前置节点的后置节点
        // 可能是当前节点,也可能不是当前节点,是其他的中断节点  
        Node predNext = pred.next;
        // 设置中断状态
        node.waitStatus = Node.CANCELLED;
				
        // 条件一: 说明出队节点是尾节点tail  , 修改tail,为前置节点
        if (node == tail && compareAndSetTail(node, pred)) {
            // 出队, pred.next设为null
            compareAndSetNext(pred, predNext, null);
        } else {
            // 状态
            int ws;
            // 当前节点不是head.next几点,也不是tail节点
            if (pred != head &&
                // 成立: 当前节点的前置节点是 需要唤醒后置节点的 不成立: 可能为0,也可能前置节点被中断了
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                // 如果状态时小于0,则修改状态为 唤醒后继节点为-1
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null)
                // if 所做的事,将pred.next节点指向 node.next节点,所以要保证pred状态为-1 signal
				 {
                // 出队节点的下一个节点
                Node next = node.next;
                // 后置节点不为空且状态为中断状态
                if (next != null && next.waitStatus <= 0)
                    // 设置pred.next节点为 node.next 
                    compareAndSetNext(pred, predNext, next);
            } else {
                // 唤醒后置节点
                unparkSuccessor(node);
            }
            // 将当前节点指向自己的next节点
            node.next = node; // help GC
        }
    }
```

尝试获取锁,判断当前队列是不是空队列,是空队列, 且不需要等待,抢占锁成功,然后将自己设为独占线程, 

不是空队列,或者抢占所失败,说明有竞争, 改锁是可重入的,判断是否是当前线程重入的,不是则返回false

抢占成功则,入队,当前队尾不为空,则直接入队尾,为空或者入队抢锁失败则进入完全入队逻辑enq,

第一次创建队列,并发自己设为head和tail,队列已存在,则for循环入队

判断当前是不是头结点的后置节点,是后置节点则可以竞争锁,不然则挂起,等待被唤醒

### Condition

```java
		
		public final void await() throws InterruptedException {
            // 判断当前线程是否被中断 ,如被中断 直接返回抛出中断异常
            if (Thread.interrupted())
                throw new InterruptedException();                                                                                                 
            // 加入condition队列
            Node node = addConditionWaiter();
            // 释放所有锁  , 如果加锁被挂起,则肯定不会被唤醒
            int savedState = fullyRelease(node);
            // 0 condition队列挂起期间 中未收到中断信号
            // -1 condition队列挂起期间 收到了中断信号
            // 1 condition队列挂起期间 未收到中断信号, 但迁移到阻塞队列时 收到了中断信号
            int interruptMode = 0;
            // isOnSyncQueue true表示已经迁移到阻塞队列
            // false 表示还未迁移,需要继续挂起
            while (!isOnSyncQueue(node)) {
								// 挂起
                LockSupport.park(this);
								// checkInterruptWhileWaiting 就算在挂起时,被中断,也会迁移到阻塞队列
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 已经进入阻塞队列了
            // 条件1 : 说明当前线程被中断过,
            // 条件2 : interruptMode != THROW_IE 成立，说明当前node在条件队列内 未发生过中断
            // 设置 interruptMode = REINTERRUPT
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // node在条件队列内时 如果被外部线程 中断唤醒时，会加入到阻塞队列，但是并未设置nextWaiter = null。
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
						
            // 条件成立：说明挂起期间 发生过中断（1.条件队列内的挂起 2.条件队列之外的挂起）
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

			// 入 condition 队列末尾
			private Node addConditionWaiter() {
            // 尾节点引用
            Node t = lastWaiter;
            // 1 当前队列已经有元素了
            // 2 node在条件队列时 CONDITION(-2)
            // t.waitStatus != Node.CONDITION 成立说明 当前node发生中断了 
            if (t != null && t.waitStatus != Node.CONDITION) {
                // 清理所有取消状态的节点
                unlinkCancelledWaiters();
                // 更新新的队尾
                t = lastWaiter;
            }
            // 创建当前线程的node节点,状态为 -2
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            // 尾节点为空
            if (t == null)
                // 说明当前队列为空,当前线程为第一个节点
                firstWaiter = node;
            else
                // 已有队列,追加队尾
                t.nextWaiter = node;
            // 设为尾节点 
            lastWaiter = node;
            return node;
        }
		
		// 清理所有取消状态的节点
		private void unlinkCancelledWaiters() {
            // 头节点
            Node t = firstWaiter;
            // 当前链表上一个正常状态的node节点
            Node trail = null;
            // 不为空
            while (t != null) {
                // 头结点的下一个节点
                Node next = t.nextWaiter;
                // 为取消状态
                if (t.waitStatus != Node.CONDITION) {
                    // 置空
                    t.nextWaiter = null;
                    // 条件成立：说明遍历到的节点还未碰到过正常节点..
                    if (trail == null)
                        //更新firstWaiter指针为下个节点就可以
                        firstWaiter = next;
                    else
                        // 让上一个正常节点指向 取消节点的 下一个节点..中间有问题的节点 被跳过去了..
                        trail.nextWaiter = next;
                    //条件成立：当前节点为队尾节点了，更新lastWaiter 指向最后一个正常节点 就Ok了
                    if (next == null)
                        lastWaiter = trail;
                }
                else //条件不成立执行到else，说明当前节点是正常节点
                    trail = t;
                t = next;
            }
        }

// 完全释放锁
final int fullyRelease(Node node) {
        // 完全释放锁是否成功  
        boolean failed = true;
        try {
            // 获取一共几把锁
            int savedState = getState();
            // 解锁 
            if (release(savedState)) {
                // 返回 当前线程返回的state值
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            // 抛出异常/解锁失败
            if (failed)
                // 设为取消状态
                node.waitStatus = Node.CANCELLED;
        }
    }
   
	// 是否在队列中
  final boolean isOnSyncQueue(Node node) {
        // 条件成立说明一定在条件队列,因为signal,迁移节点是,会先将状态改为0
        // 不成立
        // 1 node.waitStatus = 0 当前节点已经被signal了
        // 2 node.waitStatus =  1 当前线程未持有锁调用await方法,最终会将状态改为0

        // node.prev == null  signal是先修改状态,再迁移节点
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;

        // 执行到这里会出现那种情况
        // node.waitStatus != Node.CONDITION 且 node.prev != null 可以排除 node.waitStatus = 1
        // 为什么可以排除取消状态,signal不会把取消状态的节点迁移走
        // 条件成立 已经成功入队到阻塞队列 ,且当前节点后面已经有其他node了
 
        if (node.next != null) // If has successor, it must be on queue
            return true;
	      
        // 从队尾寻找与当前node相同的节点,找到返回true,未找到返回false
        return findNodeFromTail(node);
}

			// 判断是否在队列中
			private int checkInterruptWhileWaiting(Node node) {
            // 是否中断过
            return Thread.interrupted() ?
                // 这个方法只有在线程是被中断唤醒时 才会调用！
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
       }

		// 不在队列 则入队
		final boolean transferAfterCancelledWait(Node node) {
        // 成立则说明一定在CONDITION队列中
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            // 入队
            enq(node);
            // 被中断
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        // 执行到这的几中情况
        // 当前node已经被外部线程调用signal 方法迁移到阻塞队列中了
        // 同上,但是正在迁移中...
        while (!isOnSyncQueue(node))
            // 释放cpu 重试
            Thread.yield();
        // 当前节点被中断唤醒时,不在条件队列中了
        return false;
    }

		// 抛出中断异常,或者不做任何操作
		private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

		// 唤醒
		public final void signal() {
            // 如果不是当前线程直接报错
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            // 头结点引用
            Node first = firstWaiter;
            // 不为空
            if (first != null)
                doSignal(first);
        }

		// 主要唤醒方法
		private void doSignal(Node first) {
				
            do {
                // 头结点要出队,判断当前节点是否最后一个结点
                if ( (firstWaiter = first.nextWaiter) == null)
                    // 尾结点置空
                    lastWaiter = null;
                // 头结点开出队,断开引用,gc
                first.nextWaiter = null;
            // 条件1 transferForSignal(first) 是否已在阻塞队列中,
            // 条件2 `
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

			final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        // 修改当前节点的为默认状态当前节点马上就要进入阻塞队列了
        // false  挂起期间,当前node被其他线程中断唤醒过,也会进入阻塞队列
        //  await 时,未持有锁,最终线程对应的node会设置为取消状态
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        // 如阻塞队列
        Node p = enq(node);
        // 入队节点前置节点的状态
        int ws = p.waitStatus;
        // 条件1 : 是取消状态,唤醒当前节点
        // 条件2 : true,修改状态成功
        // false 当前驱node对应的线程 是 lockInterrupt入队的node时，是会响应中断的，外部线程给前驱线程中断信号之后
        // 前驱node会将状态修改为 取消状态，并且执行 出队逻辑..
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            // 状态只要不是 0 , -1 就唤醒当前线程
            LockSupport.unpark(node.thread);
        return true;
    }
```

Condition

condition条件队列,await() 进入队列,判断当前线程是否被中断,被中断则抛出异常,判断当前队列是否存在,如果存在则入队,如果不存在则创建队列,状态为-2,释放所有锁,如果加锁被挂起,则不会被唤醒,判断当前节点是否在阻塞队列,如果不在阻塞队列则需要继续park,会判断是否被中断,被中断下一个节点是否为空,如果中断之后,会直接进入阻塞队列,nextwaiter不清空,这里需要清理所有状态为取消的节点,

doSignal 唤醒第一个节点,当前节点出队,下一个节点为头结点,判断当前节点是否已经在阻塞队列中,入队到阻塞队列中,然后唤醒await住的节点