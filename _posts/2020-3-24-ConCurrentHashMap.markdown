# ConCurrentHashMap 源码

putVal ,

```java
//  onlyIfAbsent true -> 冲突时不替换原数据,  false -> 替换原数据
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
		// 扰动函数,获取hash值,使key的hashcode值的高16位也参与到下标寻址的运算中
        int hash = spread(key.hashCode());
		// 为0时 表示当前node节点为null,可以直接放元素
		// 为2时,表示当前节点可能树化为红黑树节点
        int binCount = 0;
		// 自旋等待的for循环
        for (Node<K,V>[] tab = table;;) {
			// f 表示当前的头节点  n,表示为当前map的长度,i 表示当前的要放置元素的桶为下标地址,
			// fh 表示头结点的hash值
            Node<K,V> f; int n, i, fh;
			// 判断 1: 当前tab为空,表示当前tab未初始化,后续进行初始化行为
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
			 // 判断 2: 通过tabAt计算出的桶位的头结点有没有值,没有值则尝试放入
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
				// 进行插入操作,底层c++,cas原则
				// 将tab[i]桶位的值,与期望值对比,如果一致则插入 new Node<K,V>(hash, key, value, null)
				// 这里头阶段为空,期望值也为空
				// 如果插入失败,则表示当前桶为被其他线程操作,则自旋再进行其他操作逻辑
                if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
			// 判断 3: 头结点不为空且当前桶位的头结点是FWD节点,正在扩容中
            else if ((fh = f.hash) == MOVED)
				// 当前线程帮助其他线程完成扩容任务
                tab = helpTransfer(tab, f);
			    // 判断 4: 当前桶位,为红黑树或者为链表
            else {
				// 当key值存在时,表示旧值value
                V oldVal = null;
				// 加锁,锁为当前桶位的头结点
                synchronized (f) {
					// 避免其他线程把头结点修改掉,如果已经修改掉,则加锁就已经错误
                    if (tabAt(tab, i) == f) {
						// 当前节点为普通链表
                        if (fh >= 0) {
							// key不冲突时为当前链表元素总个数,
							// 冲突时需要替换元素时,表示替换数据的下标位置(binCount - 1)
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
								// 冲突时,需要替换原位置元素,并将原value赋值并返回
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
									// put()时传参,为fasle替换元素,true是不替换元素
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
								// 不发生冲突,且为链表最后一个元素,在此追加
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,value, null);
                                    break;
                                }
                            }
                        }
						// 为treeBin 代理节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
							//默认为2,是因为最后addCount()方法需要使用到该值,值带有不同意义
                            binCount = 2;
							//条件判断: 插入红黑树节点,没有冲突则返回null有冲突则返回冲突节点
							// 不为空是则有冲突节点存在,并将冲突节点的旧值赋值并返回
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
								// 参照链表此地说法
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
				// 当前put元素位置为空,直接放置元素时为0,为链表或treeBin时不为0
                if (binCount != 0) {
					// 链表长度大与升级红黑树阈值,则升级为红黑树
					// 红黑树时为2不会进入该判断
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
					// 当前key与原有数据发生冲突,需要将其返回
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
		// 统计当前散列表一共有多少数据,是否达到扩容条件
        addCount(1L, binCount);
        return null;
    }
```

initTable  初始化tab
```java
    private final Node<K,V>[] initTable() {
    		// tab 当前tab引用  sc: 临时sizeCtl
            Node<K,V>[] tab; int sc;
    		// 判断是否未初始化
            while ((tab = table) == null || tab.length == 0) {
    			// 小于0(-1)表示其他线程正在初始化,是当前线程再次进入cpu竞争,把调度让给其他线程
                if ((sc = sizeCtl) < 0)
                    Thread.yield(); // lost initialization race; just spin
    			// 大于 0 未初始化时, 表示当前需要初始化的容量 , 已初始化,表示扩容的阈值
    			// 或者 等于 0  为默认容量值
    			// 上述为改if 的两种情况 , 符合则进行cas加锁,加锁成功之后,进入初始化阶段
                else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
    					// 二次判空,防止多线程情况下,其他线程已经初始化失败,重复初始化,丢失数据
                        if ((tab = table) == null || tab.length == 0) {
    						// sc 大于0 则表示初始化容量已经确定,否则则使用默认容量
                            int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = tab = nt;
    						// 初始化成功将 sc设为 之后扩容的阈值
                            // n无符号右移两位,相当于把值设为原来值的 1/4,
    						// 再减去,就是(3/4)n ,相当于 n * 0.75
                            sc = n - (n >>> 2);
                        }
                    } finally {
    					// 两种情况
    					// 1 : 已经加锁成功,但是发现已经不是第一次初始化,这是应该把原有的sc设置回去
    					// 2 : 初始化成功,将sc设为扩容阈值
                        sizeCtl = sc;
                    }
                    break;
                }
            }
            return tab;
        }
```
addCount() 累加数据,或者扩容判断

```java
private final void addCount(long x, int check) {
        // as ->cells 数组
        // b -> base值
        // s -> tab 中元素数量
        CounterCell[] as; long b, s;
        // cells true 数组已初始化,将数据放到cell数组内
        // false 未初始化
        if ((as = counterCells) != null ||
            // 尝试写base -> 去反后  false,写入成功,说明当前竞争不激烈,可以不用创建cell数组
            // true 写入失败,说明当前有竞争力, 需要创建cells 数组
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            // 成功进入的条件
            // true 数组已初始化,将数据放到cell数组内
            // true 写入失败,说明当前有竞争力, 需要创建cells 数组

            // a 寻址得到的cell 地址,
            // v 写入地址的期望值
            // m cell 数组的长度
            CounterCell a; long v; int m;
            // true 有竞争 false 无竞争
            boolean uncontended = true;
            // 数组未初始化
            if (as == null || (m = as.length - 1) < 0 ||
                // 寻址地址处元素为空
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                // 取反  写入base失败,发生了竞争
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                // 进行后续重试或者扩容 -> 相当于 longAdder的longAccumulate()方法
                fullAddCount(x, uncontended);
                // fullAddCount()方法处理的东西较多,不参与扩容,直接返回
                return;
            }
            // binCount传入: 
            // >=1 时,hash冲突,当前位置有一个数据,或者已经成为链表 ---- 2 时 为红黑树
            // = 0 当前桶为为空,直接放入时
            // < 时 为remove操作
            if (check <= 1)
                return;
            //获取所有元素数,期望值没参与longAdder
            s = sumCount();
        }
        // 此处是put操作的一个状态
        if (check >= 0) {
            // tab 当前链表数组
            // nt 升级的数组
            // n 当前数组的长度
            // sc sizeCtl 临时值
            Node<K,V>[] tab, nt; int n, sc;
            // 数组中的元素大于阈值,原table不为null,原数组长度没超过最大值
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 获取表示符
                int rs = resizeStamp(n);
                // 小于0时,高六位表示标识符,低六位表示参与的线程数
                if (sc < 0) {
                    // 1 : 标识符是否相等,
                  //	如果扩容完毕时,表示符肯定已变更 ps: 比如 16 -> 32,32 -> 64 标识是不一样的
                    // 2 : 扩容是否结束,(表达的意思应该是 (rs << 16) + 1)
				  // 当前线程数为1 时表示,扩容已经结束,因为第一次初始化时线程 + 2,所以为1时扩容已经结束
                    // 3 : 判断是否大于最大可有线程数 (表达意思应该是 (rs << 16 + MAX_RESIZERS))
                    // 4 : 新数组为null,扩容完毕
                    // 5 : 当前区间进度 <= 0,数组扩容完毕
                    // 2,3 存在bug,不会用到
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // 尝试帮助扩容,如果cas失败,表示竞争比较多,自旋继续执行操作,
                    // 下次进来可能已经被其他线程扩容完毕
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        // 帮助完成库容的线程,持有nextTable
                        transfer(tab, nt);
                }
                // 大于0 表示,第一次扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    // 第一次扩容不持有 nextTable
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

transfer(), 扩容,协助扩容方法

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        // n -> tab的长度
        // stride 扩容分配区间的补偿
        int n = tab.length, stride;
        // 步长计算
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 是不是第一次扩容
        if (nextTab == null) {            // initiating
            try {
                // 新创建大一倍的数组
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            // 赋值新表 和 扩容任务进度(当前tab长度)
            nextTable = nextTab;
            transferIndex = n;
        }
        // 新表长度
        int nextn = nextTab.length;
        // 设置新表为fwd节点
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 推进表示,true 表示还有桶位需要继续扩容
        boolean advance = true;
        // 当前步长扩容是否完成
        boolean finishing = false; // to ensure sweep before committing nextTab
        // 自旋扩容 
		//倒序分配区间 i -> 分配扩容区间起点,  n -> 扩容区间临界点   i <-> n 之间的元素为需要扩容
        for (int i = 0, bound = 0;;) {
            // f -> 头结点  fh -> 头结点 hash值
            Node<K,V> f; int fh;
            // 1: 给当前线程分配任务区间
            // 2: 维护当前线程任务进度
            // 3: 维护整个map的进度
            while (advance) {
                // 分配的   i -> 分配扩容区间起点,  n -> 扩容区间临界点
                int nextIndex, nextBoun d;
                // true 区间内还有任务未扩容完成
                if (--i >= bound || finishing)
                    advance = false;
                // 当前线程任务已完成,或未分配
                // true 全局扩容已经完成,扩容进度为0
                else if ((nextIndex = transferIndex) <= 0) {
                    // 设为-1,后续退出操作
                    i = -1;
                    advance = false;
                }
                // 当前线程需要分配任务区间,整个扩容还未完成
                // true,设置当前线程任务区间成功,如果剩余值小于步长,则把剩余项全部交给当前线程
                // false,线程较多,出现竞争
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    // 设置 起始点 和 临界点
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //当前线程未分配任务
            //	后两个条件是不会成立的,因为i是原tab长度减一得到的 ,nextn是原数组的两倍
            if (i < 0 || i >= n || i + n >= nextn) {
                // 临时变量 sizeCtl
                int sc;
                // 自选之后进入,扩容已经完毕,将扩容后的数组赋值给原数组
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    // 设为下一次扩容的阈值  (n << 1) - (n >>> 1) = n * 0.75
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 扩容线数 -1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 当前线程数是不是最后一个线程
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 如果是最后一个线程进行收尾工作  finishing标识扩容任务完毕
                    finishing = advance = true;
                    // 起始下标设为 数组长度
                    i = n; // recheck before commit
                }
            }
            // 如果当前头结点为空,讲此节点桶为设置为fwd节点
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 当前节点为fwd节点
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            // 当前节点不为fwd节点,且当前节点不为空,该节点为链表或者红黑树
            else {
                // 头节点加锁
                synchronized (f) {
                    // 重复判断枷锁过程中,头结点有没有别其他线程操作修改
                    if (tabAt(tab, i) == f) {
                        // ln  低位链表  hn 高位链表
                        Node<K,V> ln, hn;
                        // >= 0 为普通node链表
                        if (fh >= 0) {
                            //此地是获取最后一串连续相同的,高位数据或者低位数据,0是低位,1是高位
                            // 然后放置到新数组的高低位置
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 循环获取连续相同的高或低位数据,组成了链表结构
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            // = 0 低位链表
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            // = 1 高位链表
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            // 循环判断,除上面连续相同的高或低位数据以外的数据
                            // 并加入新的高低位链表头部,
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 写入低位链表到新tab
                            setTabAt(nextTab, i, ln);
                            // 写入低位链表到新tab
                            setTabAt(nextTab, i + n, hn);
                            // 讲旧tab  i桶位设置为fwd节点
                            setTabAt(tab, i, fwd);
                            // 继续完成后续扩容
                            advance = true;
                        }
                            // 为treeBin代理节点
                        else if (f instanceof TreeBin) {
                            // 强转为treeBin
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            // lo -> 链表  低位头节点     loTail -> 低位尾节点
                            TreeNode<K,V> lo = null, loTail = null;
                            // lo -> 链表  高位头节点     loTail -> 高位尾节点
                            TreeNode<K,V> hi = null, hiTail = null;
                            // lc,hc,计数作用,是否转为红黑树
                            int lc = 0, hc = 0;
                            // 遍历头结点
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                // 新建treeNode节点
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                // 判断高低位
                                // 低位
                                if ((h & n) == 0) {
                                    // 上一个节点为空,将此节点作为低位链表头结点
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    // 链表不为空,则将此节点插入到链表末尾
                                    else
                                        loTail.next = p;
                                    // 将p设为尾节点
                                    loTail = p;
                                    // 统计个数 +1
                                    ++lc;
                                }
                                // 高位  注释同上
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            // 小于6,将treeBin替换为普通node链表
                            // 大于6,将treeNode链表升级为红黑树
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            // 新表设置低位
                            setTabAt(nextTab, i, ln);
                            // 新表设置高位
                            setTabAt(nextTab, i + n, hn);
                            // 设置当前桶位位fwd节点
                            setTabAt(tab, i, fwd);
                            // 继续完成后续扩容
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

put时发现正在扩容要参与进来

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        // 新table, sizeCtl
        Node<K,V>[] nextTab; int sc;
        // tab不等于null, 改节点为fwd节点,且新table不为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            // 扩容标识符
            int rs = resizeStamp(tab.length);
            // 多线程情况下已被扩容完成nextTable被设为null,
            // 相等说明扩容还在进行中
            // sizeCtl < 0 表示正在扩容中
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 具体看addCount()  有解说
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

get()
```java
    public V get(Object key) {
    				//tab 当前 table
    				// e 当前key命中的桶位,p 目标节点
    				// n table 长度 
    				// eh,如果当前key命中桶位不为空,则表示已存在node的hash值 ek,则表示已存在node的key值
            Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    				// 扰动hash值,更散列
            int h = spread(key.hashCode());
    				// 数组不为空并且当前桶位不为空
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (e = tabAt(tab, (n - 1) & h)) != null) {
    						// 头结点hash值,key比较相等,直接命中返回
                if ((eh = e.hash) == h) {
                    if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                        return e.val;
                }
    						// 小于0,说明是fwd节点,或者为红黑树
                else if (eh < 0)
    								// find方法内寻找key值
                    return (p = e.find(h, key)) != null ? p.val : null;
    						// 尚书令两种都不符合,说明为普通链表,hash,equals比较即可
                while ((e = e.next) != null) {
                    if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                        return e.val;
                }
            }
            return null;
        }
```
find() → fwd find

```java
    static final class ForwardingNode<K,V> extends Node<K,V> {
            final Node<K,V>[] nextTable;
            //   创建的时候将 hash值设为 -1,将tab赋值给nextTable
            ForwardingNode(Node<K,V>[] tab) {
                super(MOVED, null, null, null);
                this.nextTable = tab;
            }
    
            Node<K,V> find(int h, Object k) {
                // loop to avoid arbitrarily deep recursion on forwarding nodes
                // 新表赋值给tab
                outer: for (Node<K,V>[] tab = nextTable;;) {
                    // e 当前元素,n tab的长度
                    Node<K,V> e; int n;
                    // 前三条件都恒不成立,最后一个表示头结点为空
                    if (k == null || tab == null || (n = tab.length) == 0 ||
                        (e = tabAt(tab, (n - 1) & h)) == null)
                        return null;


                for (;;) {
                    // 当前元素的hash
                    // 当前元素key
                    int eh; K ek;
                    // 头结点直接命中返回
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    // 可能为红黑树接口,也可能线程过多,再次进行扩容,成为fwd节点
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        // 为红黑树,执行红黑树find
                        else
                            return e.find(h, k);
                    }
                    // 普通链表,自旋再次进入逻辑
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```

remove(),  套娃的后续方法,, 在成功删除红黑树节点之后,会直接转换为node普通链表,也没有进行长度判断,正常应该为6以下转为链表

```java
final V replaceNode(Object key, V value, Object cv) {
        // 扰动hash
        int hash = spread(key.hashCode());
        // 赋值当前table
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果当前当前tab为空,或者命中头元素为空,直接返回 删除的元素不存在
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;
            // 如果头元素不为空,并且hash为 -1,表示当前fwd节点,需要参与扩容
            else if ((fh = f.hash) == MOVED)
                // 详看该方法
                tab = helpTransfer(tab, f);
                // 当前为链表,或者 treeBin
            else {
                // 命中的旧值
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    // 放置锁错对象
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            validated = true;
                            // 为普通node链表,或者node单个元素,e为循环中的node对象
                            for (Node<K,V> e = f, pred = null;;) {
                                // 当前node对象的key
                                K ek;
                                // 命中相同的key
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    // 当前节点的value
                                    V ev = e.val;
                                    // 1 : 成立则是删除操作
                                    // 2,3 : 成立时替换操作
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        // 旧值 赋值返回
                                        oldVal = ev;
                                        // 条件成立为替换操作
                                        if (value != null)
                                            e.val = value;
                                        // 删除操作,将前几点直接指向当前的节点的下一个节点
                                        // 当前节点被抛弃
                                        else if (pred != null)
                                            pred.next = e.next;
                                                // 当前为头结点,
                                        // 只需将桶节点设置为头结点的下一个节点,当前节点被抛弃
                                        else
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }
                                // 当前节点设为下一轮循环元素的前一个节点
                                pred = e;
                                 // 遍历完链表最后一个为空,跳出
                                if ((e = e.next) == null)
                                    // 想要删的值不存在
                                    break;
                            }
                        }
                        // 如果是红黑树
                        else if (f instanceof TreeBin) {
                            validated = true;
                            // 强转为红黑树节点
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            // r为根节点 , p为查找到的树节点
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                // 命中需要删除或者替换的元素
                                // 命中元素的value
                                V pv = p.val;
                                // 桶链表判断
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    // 此为替换操作
                                    if (value != null)
                                        p.val = value;
                                    // 此为删除操作,调用treeBin的删除逻辑
                                    else if (t.removeTreeNode(p))
                                        // 此地删除成功之后,直接转为普通node链表
                                        // 也没有进行进行元素判断
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                //检验表示,进入链表或者红黑树会设置为true
                if (validated) {
                    // 旧值不为空
                    if (oldVal != null) {
                          // 确定为删除操作
                        if (value == null)
                            // 总元素数 -1
                            addCount(-1L, -1);
                        // 删除替换成功之后返回旧值
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }

```

treeBin节点,  写是独占锁,读是共享锁

↓   treeBin节点元素含义

```java
static final class TreeBin<K,V> extends Node<K,V> {
        // 根节点
        TreeNode<K,V> root;
        // 链表头结点
        volatile TreeNode<K,V> first;
        // 等待者线程 -> 当lockState是读锁状态时 才存在
        volatile Thread waiter;
        // 状态读写时用,写是独占锁,读是共享锁
        // 1  写线程 -> 一个treeBin只能同时存在一个读线程
        // 2  写线程时,发现当前存在读锁,当前线程是读线程,需要等待
        // 4  每来一个线程 lockState的值加4
        volatile int lockState;
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
}

```

创建treeBin对象

```java
TreeBin(TreeNode<K,V> b) {
            // 设置hash为-2,红黑树节点,  -1 为fwd节点
            super(TREEBIN, null, null, null);
            // 头结点为当前引用节点
            this.first = b;
            // r 红黑树根节点
            TreeNode<K,V> r = null;
            // 自旋插入所有node链表数据
            for (TreeNode<K,V> x = b, next; x != null; x = next) {
                // 下一个node元素
                next = (TreeNode<K,V>)x.next;
                // 插入红黑树,左右节点设空
                x.left = x.right = null;
                // 第一次进,根节点,把头结点设为红黑树根节点
                if (r == null) {
                    // 跟节点无父节点
                    x.parent = null;
                       // 根节点为黑
                    x.red = false;
                    // 赋值给根节点
                    r = x;
                }
                else {
                    // 根节点已存在
                    // 当前节点的key
                    K k = x.key;
                    // 当前节点的hash
                    int h = x.hash;
                    // 当前节点的类
                    Class<?> kc = null;
                    // 自旋,存在自己点
                    for (TreeNode<K,V> p = r;;) {
                        // dir 表示  -1 当前节点的左节点, 1 当前节点的右节点
                        // ph,当前红黑树引用节点的hash值
                        int dir, ph;
                        // 当前红黑树节点的 key
                        K pk = p.key;
                        // 当前红黑树节点 大于 插入node节点的hash值,分配到该节点左边
                        if ((ph = p.hash) > h)
                            dir = -1;
                        // 当前红黑树节点 小于 插入node节点的hash值,分配到该节点右边
                        else if (ph < h)
                            dir = 1;
                        // 如果hash值相等,则判断类是否为空
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            // 此地会最终判断大小 , 赋值为 -1 或 1
                            dir = tieBreakOrder(k, pk);
                            // 当前红黑树节点设置为父节点
                            TreeNode<K,V> xp = p;
                            // 如果不等于null 说明存在子节点,继续自旋遍历
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            // 进来之后说明要插入的子节点位置为空
                            // 设置插入节点的父节点
                            x.parent = xp;
                            // 根据表示判断插入左右位置
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            // 插入后可能影响红黑树平衡,该方法进行平衡操作,并返回最终平衡树
                            r = balanceInsertion(r, x);
                            break;
                        }
                    }
                }
            }
            // 设置根节点
            this.root = r;
            assert checkInvariants(root);
        }

```

find()   get值的时候用到

```java
final Node<K,V> find(int h, Object k) {
            // key不为空
            if (k != null) {
                    // 循环的当前元素
	               for (Node<K,V> e = first; e != null; ) {
                    // s lockState的状态,读写区分
                    // ek 当前节点key值
                    int s; K ek;
                    // 存在线程读,将本身加入,lockState +4,
                    if (((s = lockState) & (WAITER|WRITER)) != 0) {
                        // 判断hash,key值相等,直接返回
                        if (e.hash == h &&
                            ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                            // 自旋下一个循环
                        e = e.next;
                    }
                    // 当红黑树,无其他线程操作
                    else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                                 s + READER)) {
                        // r 跟节点引用
                        // 根据key值查询出的节点
                        TreeNode<K,V> r, p;
                        try {
                            // findTreeNode() 从红黑树内查询
                            p = ((r = root) == null ? null :
                                 r.findTreeNode(h, k, null));
                        } finally {
                            Thread w;
                            // U.getAndAddInt(this, LOCKSTATE, -READER)
                            //当前线程读红黑树结束   线程退出 -4
                            // READER|WAITER = 0110 = 表示一个线程在读,一个线程在等待
                            // 且当前线程是读线程的最后一个线程
                            // w = waiter 有一个写线程在等待读线程结束
                            if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                (READER|WAITER) && (w = waiter) != null)
                                //读线程全部完毕,把等待的写线程放开
                                LockSupport.unpark(w);
                        }
                        return p;
                    }
                }
            }
            return null;
        }

```

putTreeVal

```java
final TreeNode<K,V> putTreeVal(int h, K k, V v) {
            // 这里是判断新节点放在什么位置 treeBin构造方法有写

            Class<?> kc = null;
            boolean searched = false;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if (p == null) {
                    first = root = new TreeNode<K,V>(h, k, v, null, null);
                    break;
                }
                else if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.findTreeNode(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.findTreeNode(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
                }

                // 当前节点,赋予插入节点的前一个节点
                TreeNode<K,V> xp = p;
                // 判断插入节点是否为空,不为空继续自旋向下查找
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // x 需要插入的节点,f 旧的头结点
                    TreeNode<K,V> x, f = first;
                    // 设为链表新的头结点
                    first = x = new TreeNode<K,V>(h, k, v, f, xp);
                    // 将旧的头结点,设为当前需要插入的新元素
                    if (f != null)
                        f.prev = x;
                    // 插入父节点的位置
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 是否黑色,如果是直接插入红色节点
                    if (!xp.red)
                        x.red = true;
                    else {
                        // 当前插入的父节点为红色,红红相连,需要调整平衡
                        lockRoot();
                        // 如果线程被挂起,则在读操作完毕之后,回复写线程到此处,继续执行
                        try {
                            root = balanceInsertion(root, x);
                        } finally {
                            unlockRoot();
                        }
                    }
                    break;
                }
            }
            assert checkInvariants(root);
            return null;

```

lockRoot() ,unlockRoot() 调整红黑树时加锁解锁

```java
private final void lockRoot() {
            // true 说明当前存在读锁,或者等待线程
            if (!U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER))
                // 竞争锁
                contendedLock(); // offload to separate method
        }

        /**
         * Releases write lock for tree restructuring.
         */
        // 释放锁
        private final void unlockRoot() {
            lockState = 0;
        }

        /**
         * Possibly blocks awaiting root lock.
         */
        private final void contendedLock() {
						//  
            boolean waiting = false;
            for (int s;;) {
                // 取反 ~WAITER => 1111 ... 1101  
                // (s = lockState) & ~WAITER) == 0 
                // true 表示当前没有读线程
                if (((s = lockState) & ~WAITER) == 0) {
                    // 设置写写线程  true 抢占锁成功
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                        // 设置treeBin waiter为null
                        if (waiting)
                            waiter = null;
                        return;
                    }
                }
                // WAITER 0010
                // 当前标志为 0 ,可以设置加锁设置写线程
                else if ((s & WAITER) == 0) {
                    // 设置treeBin 表示 lockState 为1  -> 写线程
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {
                     // 写线程设置成功,将此线程赋值为写线程
                        waiting = true;
                        waiter = Thread.currentThread();
                    }
                }
                // 挂起此线程
                else if (waiting)
                    LockSupport.park(this);
            }
        }

```