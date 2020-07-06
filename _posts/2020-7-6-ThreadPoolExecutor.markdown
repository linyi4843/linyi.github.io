---
layout:     post
title:      "ThreadPoolExecutor"
subtitle:   " \"ThreadPoolExecutor\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - ThreadPoolExecutor
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# ThreadPoolExecutor

线程池源码

```java
// 高3位：表示当前线程池运行状态   除去高3位之后的低位：表示当前线程池中所拥有的线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
		// 表示在ctl中，低COUNT_BITS位 是用于存放当前线程数量的位。   -> 29
    private static final int COUNT_BITS = Integer.SIZE - 3;
		// 低COUNT_BITS位 所能表达的最大数值。 000 11111111111111111111 => 5亿多。
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
		// 左移29位
		// 运行状态  111 000000000000000000  转换成整数，其实是一个负数
    private static final int RUNNING    = -1 << COUNT_BITS;
		// 正在停止 111 000000000000000000  转换成整数，其实是一个负数
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
		// 停止 000 000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;
		// 正在终止 001 000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;
		// 终止完毕 011 000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;
		大小 RUNNING  < SHUTDOWN < STOP < TIDYING < TERMINATED

    // Packing and unpacking ctl
		// 获取当前线程池运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
		// 获取当前线程池线程数量 
    private static int workerCountOf(int c)  { return c & CAPACITY; }
		//用在重置当前线程池ctl值时  会用到
    //rs 表示线程池状态   wc 表示当前线程池中worker（线程）数量 
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */
		//比较当前线程池ctl所表示的状态，是否小于某个状态s
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }
		//比较当前线程池ctl所表示的状态，是否大于等于某个状态s
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
		//小于SHUTDOWN 的一定是RUNNING。 SHUTDOWN == 0
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

		/**
     * Attempts to CAS-increment the workerCount field of ctl.
     */
    //使用CAS方式 让ctl值+1 ，成功返回true, 失败返回false
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    /**
     * Attempts to CAS-decrement the workerCount field of ctl.
     */
    //使用CAS方式 让ctl值-1 ，成功返回true, 失败返回false
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

		//将ctl值减一，这个方法一定成功
    private void decrementWorkerCount() {
        //这里会一直重试，直到成功为止。
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

		//任务队列，当线程池中的线程达到核心线程数量时，再提交任务 就会直接提交到 workQueue
    //workQueue  instanceOf ArrayBrokingQueue   LinkedBrokingQueue  同步队列
    private final BlockingQueue<Runnable> workQueue;

		//线程池全局锁，增加worker 减少 worker 时需要持有mainLock ， 修改线程池运行状态时，也需要。
    private final ReentrantLock mainLock = new ReentrantLock();

		//线程池中真正存放 worker->thread 的地方。
    private final HashSet<Worker> workers = new HashSet<Worker>();
		//条件队列
		private final Condition termination = mainLock.newCondition();

    //记录线程池生命周期内 线程数最大值
    private int largestPoolSize;

		//记录线程池所完成任务总数 ，当worker退出时会将 worker完成的任务累积到completedTaskCount
    private long completedTaskCount;

		//创建线程时会使用 线程工厂，默认使用的是 DefaultThreadFactory
    //一般不建议使用Default线程池，推荐自己实现ThreadFactory
    private volatile ThreadFactory threadFactory;

		//拒绝策略，juc包提供了4中方式，默认采用 Abort..抛出异常的方式。
    private volatile RejectedExecutionHandler handler;

		//空闲线程存活时间，当allowCoreThreadTimeOut == false 时，会维护核心线程数量内的线程存活，超出部分会被超时。
    //allowCoreThreadTimeOut == true 核心数量内的线程 空闲时 也会被回收。
    private volatile long keepAliveTime;

		//控制核心线程数量内的线程 是否可以被回收。true 可以，false不可以。
    private volatile boolean allowCoreThreadTimeOut;

		//核心线程数量限制。
    private volatile int corePoolSize;

    //线程池最大线程数量限制。
    private volatile int maximumPoolSize;

    //缺省拒绝策略，采用的是AbortPolicy 抛出异常的方式。
    private static final RejectedExecutionHandler defaultHandler =
            new AbortPolicy();
```

worker()

```java
// AQS 独占模式主要采用    state < 0 未初始化 初始化状态不能抢锁,> 0表示被占用
// ExclusiveOwnerThread 独占线程
private final class Worker
            extends AbstractQueuedSynchronizer
            implements Runnable{
        
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
				// worker内部封装的工作线程
        final Thread thread;
        /** Initial task to run.  Possibly null. */
				// 假设firstTask不为空，那么当worker启动后（内部的线程启动)会优先执行firstTask，
				// 当执行完firstTask后，会到queue中去获取下一个任务。
        Runnable firstTask;
        /** Per-thread task counter */
				// 记录完成的worker的数量
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
				// firstTask可以为null。为null 启动后会到queue中获取。
        Worker(Runnable firstTask) {
						// -1
            setState(-1); // inhibit interrupts until runWorker
						// 使用线程工厂创建了一个线程，并且将当前worker 指定为 Runnable
						// 也就是说当thread启动的时候，会以worker.run()为入口。
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
				// 主要入口
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.
				// 判断当前worker的独占锁是否被独占。
				//0 表示未被占用
        //1 表示已占用
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
				// 尝试获取锁   true抢占成功
        protected boolean tryAcquire(int unused) {
						// 修改成功
            if (compareAndSetState(0, 1)) {
								// 设置自己为独占线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
				
				// 释放锁
        protected boolean tryRelease(int unused) {
						// 独占线程 空
            setExclusiveOwnerThread(null);
						// 锁设置未占用
            setState(0);
            return true;
        }
				// 获取锁,获取失败会挂起,等待唤醒
        public void lock()        { acquire(1); }
				// 尝试获取,获取失败继续向下的逻辑
        public boolean tryLock()  { return tryAcquire(1);
				//一般情况下，咱们调用unlock 要保证 当前线程是持有锁的。
        //特殊情况，当worker的state == -1 时，调用unlock 表示初始化state 设置state == 0
        //启动worker之前会先调用unlock()这个方法。会强制刷新ExclusiveOwnerThread == null State==0
        public void unlock()      { release(1); }
				// 就是返回当前worker的lock是否被占用。
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

execute()

```java
public void execute(Runnable command) {
				// 空线程
        if (command == null)
						// 抛异常
            throw new NullPointerException();
        // 获取当前标识
        int c = ctl.get();
				// 是否超过核心线程数
        if (workerCountOf(c) < corePoolSize) {
						// 加入worker队列 并且启动成功
            if (addWorker(command, true))
								// 直接返回
                return;
						
						// 到这里上面addworker 失败了
            c = ctl.get();
        }

				// 1 -> addwirker失败
				// 2 -> 核心线程数满了
				// running状态 && 尝试将线程放入队列中
        if (isRunning(c) && workQueue.offer(command)) {
						// 放入成功了
            int recheck = ctl.get();
						// 加入队列之后,状态被其他线程修改了
						// 需要把线程从队列删掉
            if (! isRunning(recheck) && remove(command))
								// 走拒绝策略
                reject(command);
						// 1 : 是running状态
						// 2 : 不是running状态,但是remove失败
						//  担保机制,线程池处于running时 ,然是线程池中线程数为0,要保证至少有一个线程存活工作
            else if (workerCountOf(recheck) == 0)
								// 为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。
								// compareAndIncrementWorkerCount(c)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

private boolean addWorker(Runnable firstTask, boolean core) {
				// 跳出标识
        retry:
				// 自旋入队
        for (;;) {
						// 获取标识
            int c = ctl.get();
						// 当前运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
						// 条件1 : rs >= SHUTDOWN  大于等于关闭状态(SHUTDOWN < STOP < TIDYING < TERMINATED)(不是runnning状态)
						// 条件2 : ! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty())
						// 条件2.1 : rs == SHUTDOWN 是关闭状态  反向
						// 条件2.2 : firstTask == null 任务为空  反向
						// 条件2.3 : !workQueue.isEmpty() 队列中有任务 反向
						// > SHUTDOWN状态 线程不为空,队列为空
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
								// 失败
                return false;
						// 自旋
            for (;;) {
							// 线程数量
                int wc = workerCountOf(c);
								// 条件1 : 恒成立 CAPACITY  5亿多,你电脑能干五亿个线程不爆炸吗
                if (wc >= CAPACITY ||
										// core 标识, 根据核心线程和还是最大线程数
										// 线程数量已满
                    wc >= (core ? corePoolSize : maximumPoolSize))
										// 失败
                    return false;
								// 释放令牌
                if (compareAndIncrementWorkerCount(c))
										// 返回
                    break retry;
								// 线程计数器失败
								// 其他线程修改了ctl的值
                c = ctl.get();  // Re-read ctl
								// 线程池状态是否发生改变,其他线程调用shutdown,shutdownNow导致发生了改变
                if (runStateOf(c) != rs)
										// 状态发生改变,直接返回外层循环,后续判断当前线程状态是否创建线程
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
// 跳出到这里 retry

				// 创建的worker 是否启动
        boolean workerStarted = false;
				// worker 是否添加到池子中了
        boolean workerAdded = false;
				// 创建worker引用
        Worker w = null;
        try {
						// 创建worker
            w = new Worker(firstTask);
						// worker的线程
            final Thread t = w.thread;
						// 线程不为空,放置后续空指针
						// threadFactory是传入的,实现者可能出现bug
            if (t != null) {
								// 线程池全局锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
										// 线程池状态引用
                    int rs = runStateOf(ctl.get());
										// 条件1 : rs < SHUTDOWN  是running状态
										// 条件2 : (rs == SHUTDOWN && firstTask == null)
										// 2.1 : 关闭状态 2.2 线程为空
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
												// t.isAlive() 当线程start后，线程isAlive会返回true。
                        if (t.isAlive()) // precheck that t is startable
														// 已经启动  -> 有可能创建线程后直接启动(憨憨开发小哥哥)
                            throw new IllegalThreadStateException();
												// 加入线程池
                        workers.add(w);
												// 最新的线程池数量
                        int s = workers.size();
												// 大于计数器数量,则更新
                        if (s > largestPoolSize)
                            largestPoolSize = s;
												// 加入线程池成功
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
								// 加入成功
                if (workerAdded) {
										// 线程启动
                    t.start();
										// 启动成功
                    workerStarted = true;
                }
            }
        } finally {
						// 启动失败
            if (! workerStarted)
								// 出队,释放令牌
                addWorkerFailed(w);
      }
				// 返回线程是否启动成功
        return workerStarted;
    }

// 
private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
						// 已经创建
            if (w != null)
								// 从线程池中删除
                workers.remove(w);
						// 释放令牌
            decrementWorkerCount();
						//
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

final void final void runWorker(Worker w) {
				// 当前线程
        Thread wt = Thread.currentThread();
				// worker线程
        Runnable task = w.firstTask;
				// 置空,马上运行了
        w.firstTask = null;
				// 重置线程池的状态
        w.unlock(); // allow interrupts
				// 是否发生异常了  true发生异常
        boolean completedAbruptly = true;
        try {
						// 条件1 : 不为null
						// 条件2 : task = getTask() true 说明从队列里获取任务成功,getTask这个方法是一个会阻塞线程的方法
						// 如果返回null 则接下来执行结束逻辑
            while (task != null || (task = getTask()) != null) {
								// 加全局锁
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
								// 条件1 : (runStateAtLeast(ctl.get(), STOP) 判断当前线程池状态是否是STOP
								// 条件2 : (Thread.interrupted() &&
                //                runStateAtLeast(ctl.get(), STOP))) &&
                //        !wt.isInterrupted())
								// 条件2.1: Thread.interrupted() &&
                //                runStateAtLeast(ctl.get(), STOP))
								// 2.1.1 Thread.interrupted() 判断当前线程是否终端,并设置中断位false
								// 2.1.2 runStateAtLeast(ctl.get(), STOP) 再次判断 当前状态是否为STOP
								// 2.2 !wt.isInterrupted() 二次获取线程中断状态一定为fasle
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
										// 设置中断
                    wt.interrupt();
                try {
										// 钩子方法,子类实现
                    beforeExecute(wt, task);
										// 错误
                    Throwable thrown = null;
                    try {
												// 线程逻辑
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
												// 钩子方法,子类实现
                        afterExecute(task, thrown);
                    }
                }finally {
									 	// gc
                    task = null;
										// 完成数+1
                    w.completedTasks++;
										// 解锁
                    w.unlock();
										// 如果出错直接执行下面finally块
                }
            }
						// 并无异常
						// 一般getTask() 为空回来到这
            completedAbruptly = false;
        } finally {
						// 结束.退出程序
						//正常退出 completedAbruptly == false
            //异常退出 completedAbruptly == true
            processWorkerExit(w, completedAbruptly);
        }
    }
    

```

getTask()方法  

```java
	// 返回null
//1: >= STOP
//2: SHUTDOWN装填且 queue中有任务
//3: 线程数大与maximumPoolSize
//4: 线程数corePoolSize,并超时未获取到
private Runnable getTask() {
				// 默认未超时
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
						// 当前状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
						// 条件1 : 不是running 状态
						// 条件2: (rs >= STOP || workQueue.isEmpty())
						// 2.1 线程池出问题了,最低也是STOP,必须停止当前线程了
						// 2.2 2.1不成立说明当前状态肯定为SHUTDOWN状态,workQueue.isEmpty(),队列为空  返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
								// 池中线程数量-1
                decrementWorkerCount();
                return null;
            }
						// 到这里 线程池为running状态
						// 状态为SHUTDOWN状态,但是 队列不为空 
						// 线程池中 线程数量
            int wc = workerCountOf(c);

            // Are workers subject to culling?
						// 是否支持超时机制
						// true  核心线程池内线程也会被回收
						// false 维护不回收 

						// wc > corePoolSize 大于线程池核心线程数,也采用超时机制
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

						// 条件1 : (wc > maximumPoolSize || (timed && timedOut)
						// 1.1 其他线程改了maximumPoolSize的值,且比当前值小,有setmaximumPoolSize()的方法
						// 1.2: (timed && timedOut) 条件成立：前置条件，当前线程使用 poll方式获取task。上一次循环时  使用poll方式获取任务时，超时了
            //条件1 为true 表示 线程可以被回收，达到回收标准，当确实需要回收时再回收。
	
						//条件二：(wc > 1 || workQueue.isEmpty())
            //2.1: wc > 1  条件成立，说明当前线程池中还有其他线程，当前线程可以直接回收，返回null
            //2.2: workQueue.isEmpty() 前置条件 wc == 1， 条件成立：说明当前任务队列 已经空了，最后一个线程，也可以放心的退出。
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
								// 获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
								// 有则返回
                if (r != null)
                    return r;
								// 超时
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

// 线程退出
private void processWorkerExit(Worker w, boolean completedAbruptly) {
				//true 异常退出,需要ctl-1
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
						// 更新计数器
            completedTaskCount += w.completedTasks;
						// 删除当前worker
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();
			
        int c = ctl.get();
				// 说明现在为running 或者 SHUTDOWN状态
        if (runStateLessThan(c, STOP)) {
						// 非异常退出
            if (!completedAbruptly) {
								// 最小线程数
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
								// 队列不为空
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
								// 线程数大于最小值,线程够用直接返回,不够用则下面创建线程
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
						// 异常退出
						// queue 队列不为空,要保证线程正常工作,当前状态为 RUNNING || SHUTDOWN

						// 线程池线程数小于corePoolSize值,创建线程来执行
            addWorker(null, false);
        }
    }

// 退出相关
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
				// 全局加锁
        mainLock.lock();
        try {

            checkShutdownAccess();
						// 设置状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
						// 中断worker
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
						// 循环worker
            for (Worker w : workers) {
                Thread t = w.thread;
								// 未中断,且空闲的线程
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
												// 设置中断
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
								// 只释放一个空闲线程
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
						// 条件1 : isRunning(c)  running 状态
            if (isRunning(c) ||
								// 条件2 : true 大🌧等于 TIDYING
                runStateAtLeast(c, TIDYING) ||
								// 条件3: 是SHUTDOWN状态,且线程队列不为空  任务处理完再转换状态
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
						//什么情况会执行到这里？
            //1.线程池状态 >= STOP
            //2.线程池状态为 SHUTDOWN 且 队列已经空了
						// 池中线程数还大于0
            if (workerCountOf(c) != 0) { // Eligible to terminate
								// 中断一个空闲线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
								// 设置线程池状态为TIDYING状态。
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
												//设置线程池状态为TERMINATED状态。
                        ctl.set(ctlOf(TERMINATED, 0));
												// 唤醒调用 awaitTermination() 方法的线程。
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```