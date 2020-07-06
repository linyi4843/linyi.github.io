---
layout:     post
title:      "ThreadLocal"
subtitle:   " \"ThreadLocal\""
date:       2020-7-6 15:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - ThreadLocal
---

> “Yeah It's on. ”

![image1](/img/fw.jpg)

# ThreadLocal

```java
public T get() {
        Thread t = Thread.currentThread();
		// 从map中获取线程实例对象
        ThreadLocalMap map = getMap(t);
		// 已经未创建
        if (map != null) {
			// 返回entry
            ThreadLocalMap.Entry e = map.getEntry(this);
						
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
		// 初始化
        return setInitialValue();
    }
```

ThreadLocalMap 

```java
static class ThreadLocalMap {

		// 弱引用 当垃圾回收时会被回收
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        // 扩容阈值
        private static final int INITIAL_CAPACITY = 16;

        // 数组
        private Entry[] table;

        // 元素个数
        private int size = 0;

        // 扩容阈值
        private int threshold; // Default to 0

       
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        // 下一个下表的位置,如果到最后一个元素,则回到头元素
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         */
		// 前置元素 如果到第一个元素则回到末尾
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

        // 构造犯法   第一个key 和 第一个value
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
			// 初始entry 数组
            table = new Entry[INITIAL_CAPACITY];
			// 计算下标位置
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
			// 放入桶位
            table[i] = new Entry(firstKey, firstValue);
			// 当前元素为1 个
            size = 1;
			// 扩容阈值
            setThreshold(INITIAL_CAPACITY);
        }

       
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }

        
    }
```

方法

```java
// 获取entry
private Entry getEntry(ThreadLocal<?> key) {
			// 当前获取桶位
            int i = key.threadLocalHashCode & (table.length - 1);
			// 桶位元素
            Entry e = table[i];
			// 获取到直接返回
            if (e != null && e.get() == key)
                return e;
            else
				// 没获取到
                return getEntryAfterMiss(key, i, e);
        }

        // key 当前key
		// i 桶位下标
		// 当前i下标的 entry
		// 可能没有 , 也可能发生了冲突,  该map没有链表,会放在冲突位置的临近位置
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
			// 当前table表引用
            Entry[] tab = table;
			// 当前表长度
            int len = tab.length;1
            // 循环尝试获取 -> entry 为空结束循环
            while (e != null) {
                // 获取当前引用
                ThreadLocal<?> k = e.get();
                // 相等则返回
                if (k == key)
                    return e;
                // 当前entry 不为空/ 所以当前entry为过期数据,gc清理掉了
                if (k == null)
                    // 清理过期数据
                    expungeStaleEntry(i);
                else // 下一个下标
                    i = nextIndex(i, len);
                // 下一个entry
                e = tab[i];
            }
            // 循环获取完毕时真没有,返回null\
            return null;
        }
				
        // 清过期的数据 ,过期的位置下标
        private int expungeStaleEntry(int staleSlot) {
            // 当前table 引用
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            // 该数据已经是过期的了  value , entry 都置空  help gc
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            // 持有元素 -1
            size--;

            // Rehash until we encounter null
            // 当前循环的entry
            Entry e;
            // 当前下标
            int i;
            // 下一个下标的位置
            for (i = nextIndex(staleSlot, len);
                // entry == null 时 结束循环
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                // 当前key
                ThreadLocal<?> k = e.get();
                // 是过期数据
                if (k == null) {
                    // 置空元素 并且-1
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    // 优化阶段
                    // 判断当前桶位和正确桶位是否正确
                    int h = k.threadLocalHashCode & (len - 1);
                    // 如果不是正确桶位
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        // 正确桶位为空,则放在正确桶位
                        // 如果不对找一个更加临近的空桶位放置改entry
                        while (tab[h] != null)
                            // 下一个桶位
                            h = nextIndex(h, len);
                            // 正确桶位,或者 距离正确桶位更接近的桶位
                        tab[h] = e;
                    }
                }
            }
            return i;
        }

        // 谁知当前线程的对象
        private void set(ThreadLocal<?> key, Object value) {

          
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            // e,当前 entry
            // 条件 entry 为空结束循环
            for (Entry e = tab[i];
                 e != null;
              // 下一个下标
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // 替换操作,且并未过期
                if (k == key) {
                    e.value = value;
                    return;
                }
                // 已过期
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            // 当前位置为空
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash(); 
        }

       
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;
	         // 当前的过期entry下标
            int slotToExpunge = staleSlot;
            // 前一个下标,entry 为空的时候 跳出循环
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                // 获取前面 过期的对象下标
                if (e.get() == null)
                    // 记录前置过期对象下标
                    slotToExpunge = i;
            // 向后探索过期对象
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                // 找到当前对应的替换位置,且未过期
                if (k == key) {
                    // 当前的value替换掉
                    e.value = value;
                    // 把过期的entry放在当前对象
                    tab[i] = tab[staleSlot];
                    // 把正确的对象放在正确的位置
                    tab[staleSlot] = e;
                    // 前置没有找到过期数据
                    // 向后也没有找到过期数据
                    if (slotToExpunge == staleSlot)
                        // 当前位置设置为清理下标
                        slotToExpunge = i;
                    // 清理过期数据
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }
            // 向后扫描并未发现相等的key,直接把当前添加到对应位置中
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                // 下一个下标,当前expungeStaleEntry(i) 返回值下标的entry一定是个null
                i = nextIndex(i, len);
                Entry e = tab[i];
                // entry 不为空但是 get为空 说明一定为过期数据,要清理
                if (e != null && e.get() == null) {
                    n = len;
                    // 清理过数据
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }

       
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }

    
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        
```