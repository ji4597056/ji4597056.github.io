---
title: java集合启示录(四)之ConcurrentHashMap
categories: 技术
tags: [java,集合,数据结构]
date: 2018/3/24 20:00:00
---

## start

&emsp;&emsp;之前解读过的`HashMap`、`LinkedHashMap`、`TreeMap`都不是并发场景下线程安全的`Map`.这次要解读的是线程安全的`ConcurrentHashMap`,这也是各种面试中必问的java集合类之一.弄明白`ConcurrentHashMap`的设计原理,会给我们日常工作中解决多线程对共享资源的并发处理带来全新的启发.

- - -

<!--more-->

- - -

## content

### 概要

&emsp;&emsp;本文对java8的`ConcurrentHashMap`进行解读,`ConcurrentHashMap`类的注释中提到了`LongAdder`类,这是java8中新增的类,`ConcurrentHashMap`处理高并发场景问题的思想使用到了`LongAdder`的设计思想,所以会先介绍一下`LongAdder`,再进一步介绍`ConcurrentHashMap`.

### LongAdder

&emsp;&emsp;`LongAdder`是java8新增的用于并发环境的计数器,目的是为了在高并发情况下,代替`AtomicLong/AtomicInt`,成为一个用于高并发场景下的高效通用计数器.`AtomicXXX`中的CAS操作(乐观锁思想)在出现线程竞争十分激烈的情况下,会频繁地循环失败,从而导致浪费了很多cpu时间片.在这种情况下,悲观锁甚至更加高效.`LongAdder`引入了分段锁的思想,它通过维护一组按需分配的计数单元,并发计数时,不同的线程可以在不同的计数单元上进行计数,这样减少了线程竞争,提高了并发效率,是一种空间换时间的典型实践.

&emsp;&emsp;从`LongAdder`的成员变量可以看出累加器的存储变量为`base`和`cells`.

```java
    // cell数组,长度为2的N次方幂(最大值为>=cpu核数的最小2的N次方幂)
    transient volatile Cell[] cells;
	// 累积器的基本值,在两种情况下会使用：  
    // 1、没有遇到并发的情况,直接使用base,速度更快  
    // 2、多线程并发初始化table数组时,竞争失败的线程会尝试在base上进行一次累加操作
    transient volatile long base;
	// 自旋锁标识,用于扩容和创建cells
    transient volatile int cellsBusy;
```

&emsp;&emsp;为了保证`cell`的`value`的可见性,这里使用了`volatile`关键字修饰,并且为了解决**伪共享**的问题,给`Cell`类添加了`@Contended`注解(默认是无效的,需要在jvm启动时设置`-XX:-RestrictContended`).这里简单解释下伪共享:**cpu访问数据的时候,首先会尝试从缓存中读取,如果缓存未命中,则会从内存中读取.多核cpu的每个core都有自己的缓存.cpu把数据从内存加载到缓存中是以block为单位(一般大小是64字节),所以当某core缓存行中的一块数据更新时,会导致其他core对应的整个缓存行失效**.这里解决了`cells`数组在并发场景下多个`cell`(`value`)之间的伪共享问题.

```java
	@sun.misc.Contended 
	static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }
	}
```

&emsp;&emsp;`LongAdder`在累加值时的处理很复杂,关于CAS操作主要通过`Unsafe`类来执行.主要思路为在线程竞争不激烈的情况下(即CAS操作不失败)更新`base`,在线程竞争激烈的情况下更新当前线程对应的`cell`.(这里实在想吐糟一下源码,在`if`判断语句中写处理逻辑确实很高端但是很难懂!有的方法似乎刻意通过各种判断语句写的很复杂,让我产生一种错觉:似乎一些代码没有意义,本可以逻辑处理的更加清晰.当然,这很可能是我水平太低了而没看懂源码作者的用意~)

```java
 	public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
		// cells == null则直接CAS更新base
        if ((as = cells) != null || !casBase(b = base, b + x)) {
			// cells != null 或者 CAS更新base时失败
            boolean uncontended = true;
			// cells未初始化 或者 当前线程对应的cell == null 或者 CAS更新当前线程对应的cell失败
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
				// 可以理解为线程竞争激烈情况下的处理操作
                longAccumulate(x, null, uncontended);
        }
    }
	
	final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
		// 初始化线程随机数,用于索引程对应的cells中cell单元(类比于HashMap中通过key的hash值索引数组中位置)
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
		// 可以理解为扩容意向,在各种线程竞争激烈的情况下为false,此时再给一次机会,置为true,下次处理中依然竞争失败则进行扩容操作
        boolean collide = false;                // True if last slot nonempty
		// 循环直至处理成功
        for (;;) {
            Cell[] as; Cell a; int n; long v;
			// 1.cells已经初始化
            if ((as = cells) != null && (n = as.length) > 0) {
				// 1.1.对应cell为null
                if ((a = as[(n - 1) & h]) == null) {
					// 自旋锁未锁,则尝试初始化对应cell
                    if (cellsBusy == 0) {       // Try to attach new Cell
						// 先创建cell再尝试加锁(感觉这里乐观创建cell没有意义,是否可以在上一步直接尝试加锁?)
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
								// 考虑其他线程可能执行了扩容,这里重新赋值重新判断  
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
								// 释放自旋锁
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
							// 对应cell不为null,即其他线程初始化了cell,进入下一次循环
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
				// 1.2.之前发生CAS操作失败,rehash后进行下一次循环(只会在判断为true一次,以后的循环中直接进入1.3,这里相当于一次自旋.wasUncontended的设计是否多余?感觉没有太大意义)
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
				// 1.3.CAS更新cell,成功则退出循环
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
				// 1.4.上一步CAS更新cell失败,cell数组已经是最大或者中途发生了扩容操作
				// 若n >= NCPU,即永远不会进行下面的分支判断,即永远不会进行扩容操作
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
				// 1.5.collide == false则判断为true,说明进行累计操作时CAS竞争失败,则再给一次机会
				// 在下一次循环中依然CAS竞争失败,则进入1.6扩容操作
                else if (!collide)
                    collide = true;
				// 1.6.尝试加锁进行扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
						// 若其他线程未扩容则扩容
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
				// 重新给线程生成随机数,相当于降低hash冲突,可能使当前线程映射到其他cell,降低CAS失败的可能性  
                h = advanceProbe(h);
            }
			// 2.cells未初始化时则尝试加锁初始化cells
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
			// 3.尝试初始化cells失败则尝试CAS更新base
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```

&emsp;&emsp;看网上大神的博客,提到一个有意思的问题:这里的数组类型能否用`long`替换`Cell`?答案是不可以.上述源码中可以看出CAS更新`cell`和扩容`cells`是可以同时进行的,使用对象而不是基本数据类型可以利用扩容过程中数组浅拷贝依然保留旧数组中各个`cell`的引用.

&emsp;&emsp;通过计算`base`和所有`cell`单元的总和获取累加值.所以这里得到的累加值可能并非是最新值.

```java
	public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

&emsp;&emsp;最后总结一下:`LongAdder`的分段锁思想体现在线程竞争不激烈的情况下CAS更新`base`,而在竞争激烈的情况下根据cpu多核的特性,尽量使得每个cpu的core更新不同`cell`,由于`cells`数组的长度为大于等于cpu核数的最小2的N次方幂,那么数组长度是否越大越好?可以想象如果数组长度越大,每个线程映射的索引可能越分散,每个cpu的core更新`cell`产生冲突的可能越低.但是最多只会同时每一个cpu的core更新一个`cell`.每次CAS更新`cell`失败会重新计算线程随机数就是为了降低这种冲突,所以`cells`数组最大长度的限定相当于一种权衡.(这里我觉得数组最大长度可以适当放大以减少冲突,可以设置为大于cpu核数的最小2的N次方幂,例如cpu核数为8,则原先数组最大长度也为8,现在变为16)

### ConcurrentHashMap

&emsp;&emsp;回归正题,花了那么长的篇幅介绍`LongAdder`,到底和`ConcurrentHashMap`有什么联系呢?`ConcurrentHashMap`又是如何在`HashMap`的基础上建立了高性能的并发处理机制呢?

#### 成员变量

&emsp;&emsp;这里先列出`ConcurrentHashMap`部分重要的成员属性.

```java
	// 和HashMap中的table一样,代表整个哈希表
	transient volatile Node<K,V>[] table;

	// 用于哈希表扩容时的临时表,扩容完成后会被重置为null
    private transient volatile Node<K,V>[] nextTable;

	// 哈希表中节点数的base count(类比LongAdder中base)
    private transient volatile long baseCount;

	// 哈希表初始化和扩容时的重要参数.
	// 0：默认值
	// -1：代表哈希表正在进行初始化
	// 大于0：类比HashMap中的threshold,表示扩容阈值
	// 小于-1：代表有多个线程正在进行扩容,其高16位与table.length相关、低16位与当前参与扩容的线程数相关
    private transient volatile int sizeCtl;
	
	// 下一个线程领扩容任务时,分配的桶的起始索引
    private transient volatile int transferIndex;

	// 自旋锁标识(类比LongAdder中cellsBusy)
    private transient volatile int cellsBusy;

	// 哈希表中节点数的counter cells(类比LongAdder中cells)
    private transient volatile CounterCell[] counterCells;
```

#### 节点类型

&emsp;&emsp;`ConcurrentHashMap`的节点类型对比`HashMap`更加丰富.

##### **`Node`**

&emsp;&emsp;此节点代表普通链表节点,不同于`HashMap`中的`Node`,它多了一个`find()`方法,剩下4种节点都是继承该类,重写`find()`方法,旨在通过当前节点遍历查询目标key对应的节点.

&emsp;&emsp;还有一点不同是其`setValue(V value)`方法直接抛异常,因为该方法是设置新值返回旧值,要维持该方法操作的原子性,必然涉及到加锁,得不偿失,所以干脆不支持该方法.

##### **`TreeNode`**

&emsp;&emsp;此节点代表红黑树节点,和`HashMap`中的`TreeNode`基本类似,不过`ConcurrentHashMap`并不直接对此节点进行操作,而是通过`TreeBin`进行代理操作.

##### **`TreeBin`**

&emsp;&emsp;固定hash值为-2,用于代理操作`TreeNode`节点,持有红黑树的根节点.对于红黑树进行写入或删除操作时,整个红黑树的结构可能会有很大的变化,在并发场景下对读线程有很大的影响,所以`TreeBin`维护一个简单读写锁以支持线程安全操作整个红黑树.这也是`ConcurrentHashMap`不直接操作`TreeNode`的原因.

##### **`ForwardingNode`**

&emsp;&emsp;固定hash值为-1,用于辅助扩容的临时节点,维护了一个`Node<K,V>[] nextTable`引用.在扩容过程中,如果旧table的一个hash桶中全部的节点都已经迁移到新table中,则在旧table的这个hash桶中放置一个`ForwardingNode`节点.读操作时碰到该节点则将操作转发到扩容后的新table上去执行,写操作碰见该节点则尝试帮助扩容.

##### **`ReservationNode`**

&emsp;&emsp;固定hash值为-3,在`computeIfAbsent`和`compute`方法中用到,起到占位符的作用.

#### 添加(put)

&emsp;&emsp;put元素时和`HashMap`的很多处理逻辑是一样的,考虑到线程竞争的情况,为了支持并发安全,需要在循环中不断尝试put节点.这里需要注意与`HashMap`不同的是,`key`和`value`不允许为null.(和`HashTable`一样)

1. 延迟初始化table(第一次put元素时初始化)
2. 对key进行**rehash**操作.(不同的是`ConcurrentHashMap`里`(h ^ (h >>> 16)) & HASH_BITS;`,其中`HASH_BITS=0x7fffffff`,比`HashMap`多了一步`& HASH_BITS`,即保证rehash后的hash值是正数,我个人感觉这一步操作没有实际意义,因为计算索引时是hash值不管正负,与正数(table的size-1)进行与运算的结果必然为正数)
3. 循环中不断尝试put节点,遇到ForwardingNode节点,则协助扩容
4. 插入节点时判断链表还是红黑树,插入节点后若链表节点>8则链表转红黑树(此处逻辑和HashMap类似)
5. 更新size(此处逻辑和LongAdder类似)

```java

	// hash值是-1,表示这是一个forwardNode节点  
	static final int MOVED     = -1; // hash for forwarding nodes
	// hash值是-2,表示这时一个TreeBin节点 
    static final int TREEBIN   = -2; // hash for roots of trees
	
 	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
		// 操作标识,用以判断是否链表转红黑树和是否需要扩容
		// 退出循环时,binCount=2:插入(替换)红黑树节点或链表节点 >=0:插入(替换)链表节点后的原链表节点数
        int binCount = 0;
		// 循环直至put节点成功
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
			// 1.第一次put节点时初始化table,在下一次循环中put节点
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
			// 2.如果索引处无节点,尝试CAS操作put节点,成功则退出循环,此时binCount=0
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
			// 如果节点类型是ForwardingNode,则协助扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
			// 如果节点存在,且不为ForwardingNode(此时可能为Node、TreeBin)
            else {
                V oldVal = null;
				// 锁住该桶头节点,进行添加节点操作
				// 可能此时正在进行扩容且某线程想要迁移该桶内的节点,或者此时其他线程想要在该桶内添加节点
                synchronized (f) {
					// 再次检验节点是否变化,如果此前扩容线程抢占先机(也会锁住f),迁移桶内节点结束后会将该桶头节点设为ForwardingNode,此后其他被锁住的想要添加节点的线程在抢占锁后发现f已不是头节点后则进行下一次循环
                    if (tabAt(tab, i) == f) {
						// 链表(hash>=0)
                        if (fh >= 0) {
                            binCount = 1;
							// 循环结束后若binCount=8,说明插入节点后共有9个节点,此时需要扩容
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
								// 有则修改
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
								// 无则尾插
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
						// 红黑树(hash=-2)
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
				// 判断是否进行过插入节点操作(为0说明未从桶中的链表/红黑树中遍历节点,期间发生了初始化或扩容,需要进行下一次循环)
                if (binCount != 0) {
					// 插入节点后,链表节点超过8,需要链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
					// 如果原值存在,说明这次是替换value,并不增加节点,所以不会进行扩容,直接返回
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
		// 插入元素后的后续处理,1.更新size 2.判断扩容
        addCount(1L, binCount);
        return null;
    }
	
	// 初始化table
	private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
		// table未初始化完成执行操作,此时可能多个线程在同时初始化table,直到有一个线程初始化完成,其他线程退出循环
        while ((tab = table) == null || tab.length == 0) {
			// 如果sizeCtl=-1,说明有线程正在初始化table,此时让出cpu时间片
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
			// 无线程正在初始化table,则尝试置sizeCtrl=-1
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
					// 重新判断table引用(类比单例模式的双重加锁处理方式)
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
						// 重新计算阈值,等于n*0.75
                        sc = n - (n >>> 2);
                    }
                } finally {
					// 重置阈值
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
	
	// 插入节点后的后续处理 1.更新size 2.判断是否需要扩容
	// 这里的check参数理解为是否检查需要扩容,<0则不检查(因为没添加节点),<=1且竞争激烈则不检查
	private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
		// 更新size,此处逻辑类似LongAdder,不再赘述
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
			// 竞争激烈时插入链表节点时该索引处之前最多有1个节点,则直接返回
            if (check <= 1)
                return;
            s = sumCount();
        }
		// 判断是否需要扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
			// table的size(弱一致性性)达到扩容阈值(相对sizeCtl为正时而言,为负时s必然>sizeCtl)
			// table已初始化且table数组长度未达到最大值(1 << 30),依然可以继续扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
				// rs为扩容标识
                int rs = resizeStamp(n);
				// sc<0说明有线程正在扩容
                if (sc < 0) {
					// 根据sc的高16位和rs比较,判断是否为当次扩容(即未发生当次扩容已结束,又发起了新的扩容)
					// sc == rs + 1 和 sc == rs + MAX_RESIZERS 可以认为它们两个是为了处理极端的 RESIZE_STAMP_BITS = 32的情况,RESIZE_STAMP_BITS目前固定为16,所以这两个条件目前无作用
					// nextTable == null 表示扩容已结束,transferIndex<=0表示所有的桶迁移任务已经领取,此时无需协助扩容
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
					// sizeCtl + 1,表示当前协助扩容线程+1
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
				// 第一个进行扩容的线程,此时nextTable=null
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

#### 扩容

&emsp;&emsp;`ConcurrentHashMap`的扩容操作非常复杂,主要由于扩容不是原子操作,当前线程在扩容的过程中,有些线程在执行添加节点操作,有些线程再执行协助扩容操作,这就可能导致高并发场景下,当前线程在执行(n->2n)的扩容,其他线程已经在进行(2n-4n)的扩容.显然当前线程需要感知到当前扩容操作已经作废.`ConcurrentHashMap`的`sizeCtl`属性在多线程协助扩容中不仅标识了当前的线程数(低16位),还标识了当前正在处于的扩容任务(高16位).通过这个属性可以有效感知当前扩容任务是否已经作废.(虽然这种设计很巧妙,但是是否有更简单直接的方法呢?比如直接比较临时变量里的table,nextTable和成员变量里的table,nextTable的长度,若不同则能感知当前扩容任务已经作废)

&emsp;&emsp;`ConcurrentHashMap`在扩容任务进行中太多的线程来执行节点迁移,线程调度开销占比变大,反而降低了吞吐量.所以限制了每个迁移任务处理的最小桶区间(>=16).

```java
	// 帮助扩容
	final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
		// 该节点为ForwardingNode节点,说明此时正在扩容
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
				// 根据sc的高16位和rs比较,判断是否为当次扩容(即未发生当次扩容已结束,又发起了新的扩容)
				// sc == rs + 1 和 sc == rs + MAX_RESIZERS 可以认为它们两个是为了处理极端的 RESIZE_STAMP_BITS = 32的情况,RESIZE_STAMP_BITS目前固定为16,所以这两个条件目前无作用
				// transferIndex<=0表示所有的桶迁移任务已经领取,此时无需协助扩容   
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
				// 尝试扩容线程数+1,协助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
	
	// 扩容时的的标识
	static final int resizeStamp(int n) {
		// 针对每一次扩容,n不同则其高位0的数目不同,又因为其<<RESIZE_STAMP_BITS为负数(用以设置扩容时的sizeCtl)则再进行了一次|运算
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
	
	// 扩容
	private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
		// 扩容时,线程每次任务需要迁移若干桶节点
		// stride表示一次任务迁移桶的节点数量,为了防止过多线程进行扩容任务(这样会导致过多的线程上下文切换),最低任务数量为MIN_TRANSFER_STRIDE(16)
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
		// 第一个扩容线程需要初始化nextTable
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
			// 下一个扩容任务对应桶的索引
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
		// i指向当前桶,bound指向此次任务中当前线程需要处理的桶节点的区间下限
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
				// 更新 transferIndex
           		// 为当前线程分配任务,处理的桶结点区间为(nextBound,nextIndex)
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
			// 当前线程无法再接受扩容任务,即当前table所有桶迁移任务已被接收,但未必任务都已经完成
			// 如果出现 i>=n 说明当前扩容任务已经作废,会在下面判断中直接return(因为一定(sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
				// 最后一个扩容线程需要进行收尾工作
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
					// 判断下是不是本轮扩容中的最后一个线程,如果不是,则直接退出.  
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
					// 进行重新检查,从桶索引n-1至0依次检查是否全为fwd节点
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
			// 重新检查会进入这个分支
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
						// 类比HashMap,扩容后计算key.hash & oldCap
						// 结果为0则链表的节点的索引不变,结果为1则新索引为原索引+oldCap
						// ln为低位的链表放在newTable的原索引处,hn为高位的链表放在newTable的原索引+oldCap处(吐糟一下,这里好像写复杂了吧)
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
							// ln,hn其中一个是正序链表,另一个是反序链表(相对于节点在原链表中顺序而言)
							// 在新表中插入ln,hn
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
							// 在原表中插入ForwardingNode节点
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
						// 类比HashMap,红黑树节点也是跟链表一样的处理逻辑,若拆分后节点数量<=UNTREEIFY_THRESHOLD(6),则将其转换为链表
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
							// 判断是否转链表
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

#### 查询(get)

&emsp;&emsp;查询时用到了节点的`find`方法进行遍历查询(`ForwardingNode`或`TreeBin`).

```java
	public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
			// 索引处节点为链表节点,先检查头节点是否满足条件
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
			// 索引处节点非链表节点(ForwardingNode或TreeBin),此时调用该节点的find方法查询
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
			// 从链表头节点的下一个节点开始遍历查询
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

#### 总结

&emsp;&emsp;`ConcurrentHashMap`的设计目标在于尽量减少锁的力度(大量使用`CAS`操作以及`volatile`变量),合理利用CPU特性(考虑到线程调度因素以及CPU核数,多线程协助扩容).各种细节设计让人叹为观止.

### end

&emsp;&emsp;虽然很想写好这篇文章,但是水平有限,未能领会其设计思想精髓,文章写的很不理想.因为我总是感觉,有些地方是不是过度设计了.但是在反复阅读代码的过程中(因为反复看,却又反复看不懂),深深感受到细节设计的重要性.
