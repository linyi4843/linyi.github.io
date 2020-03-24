---
layout:     post
title:      "LongAdder"
subtitle:   " \"Hey Blog\""
date:       2020-3-24 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - longAdder
---

> “Yeah It's on. ”

注释迁移过来,不对等凑合看好不好


相对于 atmoic,并发性能更墙,追求最终一致性,但是单一操作没有atmoic性能好

 

它是将所有并发所要储存的值放在一个base中, ,然后进行增减工作

将并发的线程,设计为一个数组,并将所要累加的值放入base中,base是cas操作,根据数组中元素是否为空,来进行最终的增改,多线程较多情况下,还会进行扩容,依然是二的次方幂,主要采用多重判断,和手动加锁机制,来进行

## 主方法add()
```java

    public void add(long x) {
            Cell[] as; long b, v; int m; Cell a;
    				// 数组已经初始化,初始化直接走第二个条件写入数据
    				// 写入数据失败,发生了竞争或者需要扩容
            if ((as = cells) != null || !casBase(b = base, b + x)) {
    						// 默认无竞争
                boolean uncontended = true;
    
    						// 1 : 数组未初始化,需要初始化
    						// 2 : 当前要放入下标值为空,需要创建
    					  // 3 : 写入值是发生了竞争关系
                if (as == null || (m = as.length - 1) < 0 ||
                    (a = as[getProbe() & m]) == null ||
                    !(uncontended = a.cas(v = a.value, v + x)))
                    longAccumulate(x, null, uncontended);
            }
        }
```

## 核心方法longAccumulate()

```java

    // wasUncontended当前线程竞争失败才会是true
    final void longAccumulate(long x, LongBinaryOperator fn,
                                  boolean wasUncontended) {
            int h;
    				// 当前线程hash值还未分配
            if ((h = getProbe()) == 0) {
                ThreadLocalRandom.current(); // force initialization
                h = getProbe();
    						// 默认情况下放在0位置,未初始化,不当做一次竞争
                wasUncontended = true;
            }
    				// false 表示会扩容, true 可能会扩容
            boolean collide = false;                // True if last slot nonempty
            for (;;) {
    						// as 当前cell数组,cell当前需要使用的cell,n cell数组的长度,v 期望值
                Cell[] as; Cell a; int n; long v;
    						// 数组已经初始化,而且当前数组下标数据为null,需要将元素防到相应位置
                if ((as = cells) != null && (n = as.length) > 0) {
    								// 1.1 : 下标位置为空,可以进行赋值
                    if ((a = as[(n - 1) & h]) == null) {
    										// 当前处于未加锁状态
                        if (cellsBusy == 0) {       // Try to attach new Cell
                            Cell r = new Cell(x);   // Optimistically create
    												// 多线程多重判断,未加锁状态,并且获取锁
                            if (cellsBusy == 0 && casCellsBusy()) {
    														// 创建新元素是否成功
                                boolean created = false;
                                try {               // Recheck under lock
                                    Cell[] rs; int m, j;
    																//重复判断多线程情况下,该位置是否被其他线程所赋值,放置数据覆盖
                                    if ((rs = cells) != null &&
                                        (m = rs.length) > 0 &&
                                        rs[j = (m - 1) & h] == null) {
                                        rs[j] = r;
                                        created = true;
                                    }
                                } finally {
    																// 不论是否放置元素成功,都要释放锁
                                    cellsBusy = 0;
                                }
                                if (created)
                                    break;
    														// 未成功,自旋
                                continue;           // Slot is now non-empty
                            }
                        }
                        collide = false;
                    }
    								// 1.2 多线程时写入数据失败才 为false
                    else if (!wasUncontended)       // CAS already known to fail
                        wasUncontended = true;      // Continue after rehash
    								// 1.3 当前线程rehash过 线程hash值, 并且新命中的cell数组下标已经存在值
    								// true 则表示设置成功 false 则表示依然存在竞争
                    else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                                 fn.applyAsLong(v, x))))
                        break;
    								// 1.4 n大于cpu总值,不再扩容了, cells 已经被其线程扩容过了,rehash重试
                    else if (n >= NCPU || cells != as)
                        collide = false;            // At max size or stale
    								// 1.5 -> 1.4 之后近来,将该值设置true,有扩容意向
                    else if (!collide)
                        collide = true;
    								// 1.6 扩容
                    else if (cellsBusy == 0 && casCellsBusy()) {
                        try {
    												// 没有被其他线程扩容
                            if (cells == as) {      // Expand table unless stale
    														// 扩容
                                Cell[] rs = new Cell[n << 1];
                                for (int i = 0; i < n; ++i)
                                    rs[i] = as[i];
                                cells = rs;
                            }
                        } finally {
    												// 扩容是否成功都需要释放锁
                            cellsBusy = 0;
                        }
    										// 关闭扩容意向
                        collide = false;
    										// 自旋重试
                        continue;                   // Retry with expanded table
                    }
    
    								// 重置线程hash值
                    h = advanceProbe(h);
                }
    						// 初始化cell数组
                else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                    boolean init = false;
                    try {                           // Initialize table
                        if (cells == as) {
                            Cell[] rs = new Cell[2];
                            rs[h & 1] = new Cell(x);
                            cells = rs;
                            init = true;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    if (init)
                        break;
                }
    						// 其他线程正在初始化cell数组,需要将值累加到base
    						// 线程被上锁,抢占失败,需要将值累加到base
                else if (casBase(v = base, ((fn == null) ? v + x :
                                            fn.applyAsLong(v, x))))
                    break;                          // Fall back on using base
            }
        }
```
