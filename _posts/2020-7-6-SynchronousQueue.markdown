---
layout:     post
title:      "SynchronousQueue"
subtitle:   " \"SynchronousQueue\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - SynchronousQueue
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

## SynchronousQueue

```java
// true 公平队列 false 非公平队列
public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```

### 非公平队列 (TransferStack) 栈结构

```java
// cpu核数
static final int NCPUS = Runtime.getRuntime().availableProcessors();

    /**
     * The number of times to spin before blocking in timed waits.
     * The value is empirically derived -- it works well across a
     * variety of processors and OSes. Empirically, the best value
     * seems not to vary with number of CPUs (beyond 2) so is just
     * a constant.
     */
    // 核心数为1 不能自旋
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

    /**
     * The number of times to spin before blocking in untimed waits.
     * This is greater than timed value because untimed waits spin
     * faster since they don't need to check times on each spin.
     */
    // 自旋次数
    static final int maxUntimedSpins = maxTimedSpins * 16;

    /**
     * The number of nanoseconds for which it is faster to spin
     * rather than to use timed park. A rough estimate suffices.
     */
    // 超时时间 纳秒
    static final long spinForTimeoutThreshold = 1000L;
```

```java
E transfer(E e, boolean timed, long nanos){
						
            SNode s = null; // constructed/reused as needed
            // e为则表示是 REQUEST请求 不为空则表示 DATA 请求
            int mode = (e == null) ? REQUEST : DATA;

            for (;;) {
                // 栈顶
                SNode h = head;
                // true 头为空  
                // true 表示为相同的请求
                if (h == null || h.mode == mode) {  // empty or same-mode
                    // 条件1 : true 设置了超时限制
                    // 条件2 : true 不支持阻塞等待
                    if (timed && nanos <= 0) {      // can't wait
                        // 条件1 : 不为null
                        // 已被中断
                        if (h != null && h.isCancelled())
                            // 这只下一个为栈顶元素
                            casHead(h, h.next);     // pop cancelled node
                        else//大部分情况从这里返回。
                            return null;
                    // 栈顶为空 或者 式与当前请求一致，且当前请求允许 阻塞等待。
                    // cas 入栈操作
                    } else if (casHead(h, s = snode(s, e, h, mode))) {
                        // 入站成功了,等待被匹配
                        // 匹配的节点
                        SNode m = awaitFulfill(s, timed, nanos);
                        // 是节点本身,为取消状态  出栈
                        if (m == s) {               // wait was cancelled
                            clean(s);
                            return null;
                        }
                        //执行到这里 说明当前Node已经被匹配了...

                        //条件一：成立，说明栈顶是有Node
                        //条件二：成立，说明 Fulfill 和 当前Node 还未出栈，需要协助出栈。
                        if ((h = head) != null && h.next == s)
                            // 协助出栈
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                //什么时候来到这？？
                //栈顶Node的模式与当前请求的模式不一致，会执行else if 的条件。
                //栈顶是 (DATA  Reqeust)    (Request   DATA)   (FULFILLING  REQUEST/DATA)
                //CASE2：当前栈顶模式与请求模式不一致，且栈顶不是FULFILLING
                } else if (!isFulfilling(h.mode)) { // try to fulfill
                    // 取消状态
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    // 正常状态,cas入栈成功
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
												
                        for (;;) { // loop until matched or waiters disappear
														
                            SNode m = s.next;       // m is s's match
                            // 超时或者被中断为空 clean掉了
                            if (m == null) {        // all waiters are gone
                                // 清空栈
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                             //尝试匹配
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                // 匹配成功出栈
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.i
                                //当前NODE模式为REQUEST类型：返回匹配节点的m.item 数据域
                                //当前NODE模式为DATA类型：返回Node.item 数据域，当前请求提交的 数据etem);
	                            } else                  // lost match
                                // 强制出栈(循环匹配设置当前节点的下一个节点) 
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                // 来到这是正在匹配的状态
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    // //当s.next节点 超时或者被外部线程中断唤醒后，会执行 clean 操作 将 自己清理出栈，此时
                    //站在匹配者线程 来看，真有可能拿到一个null。
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        // 匹配
                        if (m.tryMatch(h))          // help match
                            //双双出栈，让栈顶指针指向 匹配节点的下一个节点。
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                        // 强制出栈(循环匹配设置当前节点的下一个节点)
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
```

awaitFulfill()

```java
// 正常返回匹配节点
// 不正常返回自己
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
          // 超时时间
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            // 要自旋的次数
            int spins = (shouldSpin(s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            for (;;) {
                // 被中断
                if (w.isInterrupted())
                    s.tryCancel();
                // 匹配的节点
                SNode m = s.match;
                // 匹配了
                if (m != null)
                    return m;
                // 有超时时间
                if (timed) {
                    // 超时时间
                    nanos = deadline - System.nanoTime();
                    // 超时了
                    if (nanos <= 0L) {
                        s.tryCancel();
                        continue;
                    }
                }
                // 自旋 次数-1
                if (spins > 0)
                    spins = shouldSpin(s) ? (spins-1) : 0;
                // 自旋结束了,设置线程,等待被挂起
                else if (s.waiter == null)
                    s.waiter = w; // establish waiter so can park next iter
	                else if (!timed) // 挂起
                    LockSupport.park(this);
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }

boolean shouldSpin(SNode s) {
            SNode h = head;
            // 当前是栈顶
            // 为null 说明,自旋的时候,来了个线程取走了,双双出队
            // 不是栈顶元素,且当前正在匹配中
            return (h == s || h == null || isFulfilling(h.mode));
        }
```

clean()

```java
void clean(SNode s) {
            // 清除当前节点
            s.item = null;   // forget item
            s.waiter = null; // forget thread
            // 下一个节点
            SNode past = s.next;
            // 下一个节点不为空,并且不是自己
            // 当前节点的下一个节点正常/切换下下节点
            if (past != null && past.isCancelled())
                // 切到下下一个节点
                past = past.next;

            // Absorb cancelled nodes at head
            // 栈头
            SNode p;
            // true 不为空
            // 清除全部取消状态的节点
            while ((p = head) != null && p != past && p.isCancelled())
                casHead(p, p.next);

            // Unsplice embedded nodes
            // 清除掉队列中间的取消状态节点,节点指向,取消状态节点的下一个节点
            // 从而使取消状态节点出队
            while (p != null && p != past) {
                SNode n = p.next;
                if (n != null && n.isCancelled())
                    p.casNext(n, n.next);
                else
                    p = n;
            }
        }
```

### 公平模式(TransferQueue)

```java
static final class TransferQueue<E> extends Transferer<E> {			
      // 下一个node			
      volatile QNode next;          // next node in queue
      // 有数据 data类型   == null request类型
      volatile Object item;         // CAS'ed to or from null
      // 未匹配则会被挂起,把当前线程存在 waiter
      volatile Thread waiter;       // to control park/unpark
      // true data 类型 false request类型
      final boolean isData;

			/** Head of queue */
     // 头节点
      transient volatile QNode head;
			
      /** Tail of queue */
      // 尾结点
      transient volatile QNode tail;

            /**
     * Reference to a cancelled node that might not yet have been
     * unlinked from queue because it was the last inserted node
     * when it was cancelled.
     * 表示被清理节点的前驱节点。因为入队操作是 两步完成的，
     * 第一步：t.next = newNode
     * 第二步：tail = newNode
     * 所以，队尾节点出队，是一种非常特殊的情况，需要特殊处理
     */
    transient volatile QNode cleanMe;
}

```

```java
E transfer(E e, boolean timed, long nanos) {
	QNode s = null; // constructed/reused as needed
            // 不为空是data请求
            boolean isData = (e != null);

            for (;;) {
                // 头尾引用
                QNode t = tail;
                QNode h = head;
                // 自选等待初始化数据
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin
                // true 只有当前一个节点,说明是空队列
                // 条件2 : true 尾结点和当前节点状态相等,不能匹配只能入队
                if (h == t || t.isData == isData) { // empty or same-mode
                    // 尾结点的下一个节点
                    QNode tn = t.next;
                    // false 说明有其他线程修改了
                    if (t != tail)                  // inconsistent read
                        continue;
                    // tail 有其他线程并发提前入队了
                    if (tn != null) {               // lagging tail
                        // 协助更新多线程的新的尾结点的下一个结点为新的tail结点
                        advanceTail(t, tn);
                        continue;
                    }
                    // 不支持阻塞等待
                    if (timed && nanos <= 0)        // can't wait
                        return null;
                    // 未创建node , 
                    if (s == null)
                        // 创建新node
                        s = new QNode(e, isData);
                    // 设置当前为尾结点的下一个节点失败
                    if (!t.casNext(null, s))        // failed to link in
                        // 继续设置
                        continue;
                    // 到这里说明尾结点下一个节点成功,谁知自己为tail 节点
                    advanceTail(t, s);              // swing tail and wait
                    // 返回匹配的节点
                    Object x = awaitFulfill(s, e, timed, nanos);
                    // 是自己, 说明是取消状态
                    if (x == s) {                   // wait was cancelled
                        // 清理
                        clean(t, s);
                        return null;
                    }

                    //执行到这里说明 当前Node 匹配成功了...
                    //1.当前线程在awaitFulfill方法内，已经挂起了...此时运行到这里时是被 匹配节点的线程使用LockSupport.unpark() 唤醒的..
                    //被唤醒：当前请求对应的节点，肯定已经出队了，因为匹配者线程 是先让当前Node出队的，再唤醒当前Node对应线程的。

                    //2.当前线程在awaitFulfill方法内，处于自旋状态...此时匹配节点 匹配后，它检查发现了，然后返回到上层transfer方法的。
                    //自旋状态返回时：当前请求对应的节点，不一定就出队了...

                    // 是否在队列内
                    if (!s.isOffList()) {           // not already unlinked
                        // 已经出队,把当前节点设为头结点,既出队
                        advanceHead(t, s);          // unlink if head
                        // 当前为REQUEST请求,且匹配到了DATA
                        // 但是为取消状态,把自己的域只想自己出队
                        if (x != null)              // and forget fields
                            s.item = s;
                        // 出队线程置空
                        s.waiter = null;
                    }
                    // true REQUEST请求,匹配到了对应DATA
                    // fasle DATA 匹配到了对应REQUEST 域置空
                    return (x != null) ? (E)x : e;

                } else {                            // complementary-mode
                    // 和队尾的数据类型不一样,可以进行匹配
										
                    // 真正的头结点
                    QNode m = h.next;               // node to fulfill
                    // 并发时候别的线程抢占了先机
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read
                    // 真正头节点的域
                    Object x = m.item;
                    // 条件1 : isData == (x != null)
                    // true DATA false REQUEST   基本不成立,进来的都是不同类型的数据
                    // 条件2 : true 取消状态
                    // 条件3 : DATA数据,设置域为空,已经匹配过了
                    //					REQUEST,设置域为匹配到的DATA域内的数据
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        // 设置头出队
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }
                    // 设置头出队
                    advanceHead(h, m);              // successfully fulfilled
                    // 匹配成功 , 唤醒匹配的节点
                    LockSupport.unpark(m.waiter);
                    return (x != null) ? (E)x : e;
                }
            }
        }

// 返回正确匹配节点
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            /* Same idea as TransferStack.awaitFulfill */
            // 超时时间
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            // 当前线程
            Thread w = Thread.currentThread();
            // 需要自选的次数
            int spins = ((head.next == s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            // 自旋
            for (;;) {
                // 线程中断
                if (w.isInterrupted())
                    // 设置节点为取消装填
                    s.tryCancel(e);
                // 数据域
                Object x = s.item;

                // 当节点为DATA类型时, x为当前节点的数据域, e 为当前节点创建时的数据域
                // 1 : item = null 说明点前节点和REQUEST节点匹配成功
                // 2 : item != null 且 != this 表示要传递的数据
                // 3 : item == this 取消状态

                // 当节点为REQUEST时
                // 1 : item != null 且 != this 已经匹配到节点
                // 2 : item == null ,未匹配
                // 3 : item == this 取消状态
                
                // true
                // DATA :       REQUEST: 
                // 情况 1,3      情况: 1,3 
				
                if (x != e)
                    return x;
								
                // 是否支持超时机制
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        // 超时设为取消状态
                        s.tryCancel(e);
                        continue;
                    }
                }
                // 是否还可以自旋
                if (spins > 0)
                    --spins;
                // 当前waiter 为空,设置当前线程,方便挂起之后唤醒
                else if (s.waiter == null)
                    s.waiter = w;
                // 不支持超时机制,普通挂起
                else if (!timed)
                    LockSupport.park(this);
                // 支持超时机制,超市挂起机制,且超时时间太小的话则不挂起,不如自旋节省资源
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }

// pred 尾结点,s 请求节点
void clean(QNode pred, QNode s) {
            // 是清除线程 线程置空
            s.waiter = null; // forget thread
	          
            // 得到出队节点位置
            while (pred.next == s) { // Return early if already unlinked
                // 头
                QNode h = head;
                // 真正的头
                QNode hn = h.next;   // Absorb cancelled first node as head
                // 头结点不为空 且 是取消状态
                if (hn != null && hn.isCancelled()) {
                    // 出队
                    advanceHead(h, hn);
                    continue;
                }
                // 尾结点
                QNode t = tail;      // Ensure consistent read for tail
                // dummy结点 无需清理
                if (t == h)
                    return;
                // 尾结点的下一个节点
                QNode tn = t.next;
                if (t != tail)
                    continue;
                if (tn != null) {
                    advanceTail(t, tn);
                    continue;
                }
                // 到这里说明上述条件都不成立
                // 出队节点的上一个节点,指向出队节点的下一个节点,出队节点出队完毕
                if (s != t) {        // If not tail, try to unsplice
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }

                QNode dp = cleanMe;
                // 第二次进入 dp != null
                if (dp != null) {    // Try unlinking previous cancelled node
                    // 获取要删除的节点
                    QNode d = dp.next;
                    // 被删除节点的下一个节点
                    QNode dn;
                    // 条件1 : d == null 节点已被删除
                    if (d == null ||               // d is gone or
                        // 条件2 : d == dp ,dummy节点
                        d == dp ||                 // d is off list or
                        // 条件3 : !d.isCancelled() 被删除节点状态正常
                        !d.isCancelled() ||        // d not cancelled or
                        //d != t && (dn = d.next) != null && dn != d && dp.casNext(d, dn))
                        // 条件4 : 被删除节点不是尾结点 且 被删除节点的下一个节点不为空 且 不是相同节点 最后出队成功
                        // 删除节点前驱节点指向,删除节点后一个节点
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        // cleanMe置空
                        casCleanMe(dp, null);  // 再次自旋执行到 else if 设置cleanMe
                        //未改变  前置节点等于  cleanMe 返回,
                    if (dp == pred)
                        return;      // s is already saved node
                        // 为空,ckeanMe 设置为出队节点的前驱节点
                } else if (casCleanMe(null, pred))
                    return;          // Postpone cleaning s
            }
        }
```