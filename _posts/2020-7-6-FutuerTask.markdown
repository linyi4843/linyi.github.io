---
layout:     post
title:      "FutuerTask"
subtitle:   " \"FutuerTask\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - FutuerTask
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# FutuerTask

Runnable  → 仅仅代表的是一个可执行的业务逻辑

Future 线程句柄,用来get或者cancel

### 变量

```java
		// 任务的执行状态
    private volatile int state;
		// 当程尚未执行
    private static final int NEW          = 0;
		//当前正在结束,但尚未结束
    private static final int COMPLETING   = 1;
		//当前已经结束
    private static final int NORMAL       = 2;
		//当前抛出了异常
    private static final int EXCEPTIONAL  = 3;
		//当前任务被取消
    private static final int CANCELLED    = 4;
		//当前任务中断中
    private static final int INTERRUPTING = 5;
		//当前任务已经中断 
    private static final int INTERRUPTED  = 6;

		// submit() -> Runnale/Callable runnable使用装饰器模式伪装成callable
    /** The underlying callable; nulled out after running */
    private Callable<V> callable;

    /** The result to return or exception to throw from get() */
		// 正常情况: 线程结束表示线程的执行结果
		// 非正常情况: 保存callable向上抛出的异常
    private Object outcome; // non-volatile, protected by state reads/writes

    /** The thread running the callable; CASed during run() */
		// 当前任务被线程执行,保存当前线程的对象引用
    private volatile Thread runner;

    /** Treiber stack of waiting threads */
		// 会有很多线程去get当前线程的结果,这里采用stack数据结构来保存,头插,头取的队列
    private volatile WaitNode waiters;
```

### 构造方法

```java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
				// 开发者传入的线程业务逻辑
        this.callable = callable;
				// 状态设置为new
        this.state = NEW;       // ensure visibility of callable
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
				// 通过装饰着模式,设置为callbale,get返回值可能为 null,也可能为传入的result
				// 一般是传入什么返回什么,没传则返回null
        this.callable = Executors.callable(runnable, result);
				// 设置状态
        this.state = NEW;       // ensure visibility of callable
    }
```

### 任务执行入口

```java
// 线程执行方法 -> submit -> newTaskfor -> excute -> pool
				// 判断当前线程是不是已经运行过,或者cas失败,存在竞争
public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
				// 未运行和cas成功
        try {
						// 开发者传入的业务逻辑线程
            Callable<V> c = callable;
							// 任务线程是否为空并且未运行过(防止被其他线程cancel掉)
            if (c != null && state == NEW) {
                V result;
								// 是否抛出异常
                boolean ran;
                try {
										// 运行线程任务,并返回结果
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
										// 开发者写得逻辑出现问题
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
						// 将正在执行的任务线程设为null
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

		//出现异常,设置线程状态
		protected void setException(Throwable t) {
				// cas设置正在结束
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
						// 将异常赋值给outcome
            outcome = t;
						// cas设置任务出现异常
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

	// run方法正常结束
	protected void set(V v) {
				// cas设置线程正在结束
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
						// 把正常运行的结果返回
            outcome = v;
						//设置线程正常运行结束
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

	//线程完成
	private void finishCompletion() {
        // assert state > COMPLETING;
				// 队列中存在未运行的线程
        for (WaitNode q; (q = waiters) != null;) {
						// cas 设置
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
								// 自旋
                for (;;) {
										// 队列中的任务
                    Thread t = q.thread;
										// 任务不为空
                    if (t != null) {
												// 置空 help gc
                        q.thread = null;
												// 唤醒挂起的线程
                        LockSupport.unpark(t);
                    }
										
                    WaitNode next = q.next;
										// 如果为空跳出,表示当前为最后一个节点
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
										// 不为空自旋记性后续
                    q = next;
                }
                break;
            }
        }

        done();
				// help gc
        callable = null;        // to reduce footprint
    }
```

### get

```java
	//获取当前线程的返回结果
	public V get() throws InterruptedException, ExecutionException {
				// state 引用
        int s = state;
				// 线程状态是正在结束或者未运行
        if (s <= COMPLETING)
						// 等待线程完成
            s = awaitDone(false, 0L);
        return report(s);
    }
	
	// 等待线程完成
	private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
				// 是否有超时限制
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
				// 等待队列
        WaitNode q = null;
				// 当前线程有没有入队,
        boolean queued = false;
				// 自旋
        for (;;) {
						// 判断当前线程是否是中断进来的
            if (Thread.interrupted()) {
								// 中断进入,删除当前节点,并抛出异常
                removeWaiter(q);
                throw new InterruptedException();
            }
						// 获取最新的state状态
            int s = state;
						// true 说明线程已经完成  结果未知  正常完成/抛出已成/中断
            if (s > COMPLETING) {
								// 已经入队,删除当前节点
                if (q != null)
                    q.thread = null;
								// 返回完成的状态
                return s;
            }
						// true 说明正在完成
            else if (s == COMPLETING) // cannot time out yet
								// 让出cpu,继续进行cpu竞争, 因为正在完成的状态基本秒过,时间很短
                Thread.yield();
						// 第一次为空,先创建node
            else if (q == null)
                q = new WaitNode();
						// 设置入队 
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
						// 挂起当前任务线程
            else
                LockSupport.park(this);
        }
    }

	// 删除操作
	private void removeWaiter(WaitNode node) {
				// 一般成立
        if (node != null) {
						// 当前线程设为null
            node.thread = null;
						// 循环跳出点
            retry:
            for (;;) {          // restart on removeWaiter race
								// 节点的前前引用      q -> 任务队列.....
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
										// 当前节点的下一个节点
                    s = q.next;
										// 当前节点的线程不为空
                    if (q.thread != null)
												// 设置头线程
                        pred = q;
										// 来到这里说明当前线程 为空,判断他的头结点是否为空
                    else if (pred != null) {
												// 如果不为空就把当前节点置空,将上一个节点指向下一个节点
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
										// 说明当前节点是头结点,下一个节点设为头结点
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

	// 返回结果
	private V report(int s) throws ExecutionException {
				// 正常表示 结果值
				// 非正常 表示异常
        Object x = outcome;
				// 正常结束
        if (s == NORMAL)
            return (V)x;
				// 被取消中断
        if (s >= CANCELLED)
            throw new CancellationException();
				// 开发者业务异常
        throw new ExecutionException((Throwable)x);
    }
```

### 中断

```java
	// 中断
	public boolean cancel(boolean mayInterruptIfRunning) {
				// 当前线程处于未运行状态
				// cas设置当前线程为中断中
				// !fasle  设置失败直接返回设置失败
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
				// 修改状态成功则进行下一步操作
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
										// 当前线程不为空
                    Thread t = runner;
                    if (t != null)
												// 线程中断
                        t.interrupt();
                } finally { // final state
										// 设置线程中断完毕
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
						// 唤醒get线程
            finishCompletion();
        }
        return true;
    }
```

ps:

创建线程池: 

会吧Rannable通过装饰器模式转为callbale

submit再把callbale 转换为 futureTask对象

如果当前线程数没有达到核心线程数,会直接创建新线程来运行

如果达到则放入队列等待

执行时会调用run方法,执行成功则返回result的值,执行失败业务错误返回错误值,并设置state状态,并唤醒正在等待的get的值,finishcomplete方法

get方法是获取线程的执行结果,如果当前线程运行完毕则直接返回值,如果未运行完毕则等待,挂起当前线程,并且加入等待队列,或者线程被中断直接抛出异常,主要方法awitDone(),此方法会判断,当前线程的状态,相应不同的状况

cancel方法,是中断线程,原因当前下线程在一段时间未响应,不知道到底对错,此时可中断线程,设置线程执行状态,并且删除等待线程队列,finishcomplete方法并且唤醒get线程

线程池在使用的时候

使用submit excute

会把runnable 采用装饰器模式变为callbale然后会将callable转为futruetak对象

然后偶先判断当前的线程池是否达到核心线程数如果未达到创建新的,如果达到会进行入队然后等待其他线程完成,运行线程时会判断当前线程的状态,是否未运行过过着被cancel掉,如果满足满足就执行业务逻辑,
不满足就说明运行过或者被cancel,执行退出逻辑,放空当前线程返回结果,正常之心中,业务正常则,返回结果,业务抛出异常则返回异常,并且,修改状态值,如果有get的阻塞方法,则唤醒阻塞的方法