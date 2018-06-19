---
title: java集合启示录(二)之HashMap、LinkedHashMap
categories: 技术
tags: [java,集合,数据结构]
date: 2018/3/12 20:00:00
---

## start

&emsp;&emsp;说起最常用的集合类,可能大家第一时间想到的是`ArrayList`和`HashMap`.在启示录(一)中已经分析了`ArrayList`,顺序添加和随机访问是它的优势,而插入和删除是它的短板.那么今天就来研究一下`HashMap`这个非常典型的数据结构吧.

- - -

<!--more-->

- - -

## content

### 概要

&emsp;&emsp;本文主要解读两种`Map`:`HashMap`和`LinkedHashMap`.前者我们经常使用,但是细读源码发现它的实现细节远比想象中的复杂.后者虽然不算常用,但也有其独特使用场景.

### HashMap

&emsp;&emsp;`HashMap`是一种基于键值对（K-V）形式的存储结构.脑补一下设计`HashMap`:使用数组来存储数据,计算key的`hashCode`作为hash值,根据hash值与数组length进行取模运算,得到的余数即为该key在数组中的`index`.此时有可能出现多个不同的key计算出同一个`index`(即出现了hash碰撞).所以存放在数组中的元素应为一个链表的头节点.在java8之前的`HashMap`确实是基于**数组+链表**来实现.java8中对`HashMap`做的很大的改进,采用**数组+链表+红黑树**实现.

- Key和Value都允许为空,Key重复会覆盖、Value允许重复
- 无序,即遍历`HashMap`的时候,得到的元素的顺序基本不可能是`put`的顺序
- 非线程安全,拥有`fail-fast`机制

#### 结构说明

&emsp;&emsp;在介绍`HashMap`的内部结构原理前,先来看下`HashMap`的常量属性.

```java
	// 默认数组初始化容量
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
	// 数组最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
	// 默认负载因子	
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	// 链表->红黑树的阈值
    static final int TREEIFY_THRESHOLD = 8;
	// 红黑树->链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
	// 最小树化数组容量
    static final int MIN_TREEIFY_CAPACITY = 64;
```

&emsp;&emsp;`HashMap`在初始化时可以传递两个参数:初始化容量(`initialCapacity`)、负载因子(`loadFactor`),若没传则会使用默认值`DEFAULT_INITIAL_CAPACITY`(16)、`DEFAULT_LOAD_FACTOR`(0.75f).

```java
 	public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
		// 初始化对象时并没有直接初始化table数组,这是一种延迟加载的思想
		// 当第一次向HashMap中put元素时再初始化数组
		// 这里其实设计的很巧妙,并没有保存initialCapacity,在后文resize()方法中可以看出:
		// 初始化时选用threshold作为table数组的size,并且重新设值threshold=threshold*loadFactor
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

- **`initialCapacity`**:指存储数据的数组的初始化容量,**`HashMap`会根据`initialCapacity`的值重新计算出一个大于等于该值的2的N次方幂作为数组的初始化大小**,比如`initialCapacity`=9,则`tableSizeFor(initialCapacity)`=16.
- **`loadFactor`**:用于计算数组`resize`的阈值(`threshold`),**threshold=tableSizeFor(initialCapacity)*loadFactor**,比如`initialCapacity`=4,`loadFactor`=0.75f,则向`HashMap`put第四个元素时(> 4*0.75)会触发`resize`操作.这里注意`loadFactor`可以设置为>=1的值.

##### 数组容量限定

&emsp;&emsp;`HashMap`内部数组的容量被限定为2的N次方幂,每次扩容时也是扩充为原容量的两倍,主要因为`HashMap`根据key的hash值对数组的大小取模得到在数组的位置,使用了`&`运算符(**tab[(tab.length - 1) & hash]**),这个取模操作的正确性依赖于length必须是2的N次幂.同时由于int类型的数据范围为:-2^31 ~ (2^31)-1,这也就限定了数组的最大容量为2^30,因为对于2^30继续扩容结果为2^31,已经溢出int类型的数据范围了.

```java
	// 根据传入的initialCapacity,计算出大于等于该值的最小的2的N次方幂
	// 高位右移,不断补1
	static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

##### 计算hash值

&emsp;&emsp;不管增加、删除、查找键值对,根据key的hash值定位到在数组的位置这一步都至关重要.`HashMap`在计算key的hash值时,并没有直接使用key对象自身的`hashCode`,而是根据key的`hashCode`,让其高位和低位进行异或计算得出一个新的hash值.

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

&emsp;&emsp;为何要大费周章重新计算hash值呢?上文提到计算索引的操作为`tab[(tab.length - 1) & hash]`,其中`tab.length`正好相当于一个低位掩码.`&`操作的结果就是散列值的高位全部归零,只保留低位值,用来做数组下标访问.以数组初始长度为16为例,16-1=15,二进制表示是`00000000 00000000 00001111`.让其与某散列值做`&`操作,结果就是截取了最低的四位值.

```
	10101010 10101010 00100101
&	00000000 00000000 00001111
----------------------------------
	00000000 00000000 00000101
```

&emsp;&emsp;可以看出`hashCode`的高位并没有参与计算索引时的运算,这种情况下可能会由于不太好的散列函数而导致出现严重的hash碰撞.所以`HashMap`就采用了权衡之后的方案,通过引入性能较好的**扰动函数**,通过对`hashCode`与其高16进行异或计算,混合原始哈希码的高位和低位,以此来加大低位的随机性,从而减少hash碰撞的概率.虽然重新计算hash值需要耗费多余的CPU计算,但是对比其带来的性能优化还是值得的.

#### 添加或更新操作

&emsp;&emsp;`HashMap`的增、删、查、改的操作流程远比之前介绍的`List`复杂的多,这里只详细分析一下添加或更新元素的方法,其余方法可自行查看源码.为了方便说明.这里从美团点评技术团队博客中盗了一份HashMap的put方法执行流程图.

![HashMap put方法执行流程图](http://okzr61x6y.bkt.clouddn.com/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

```java
	//onlyIfAbsent控制已存在key的行为,若为true,则key存在时不修改
	//evict用于控制插入回调函数的行为(是否要剔除元素保证数组总大小不变),序列化时调用evict为false,其余为true
	//evict在HashMap中并无作用,主要设计给其子类LinkdedHashMap使用,后文会说明
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		// 若当前table还没有分配空间,或大小为0,则先用resize()初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		// 若key对应数组的index无Node,则直接存放该Node
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
		// 已存在Node
        else {
            Node<K,V> e; K k;
			// 若根节点为要插入的key
			// 注意这里的判断条件 1.两个对象hash值相等 2.两个对象equals相等
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
			// 若该节点为红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
			// 若该节点为链表
            else {
				// 遍历链表
                for (int binCount = 0; ; ++binCount) {
					// 未找到key,则尾节点插入Node
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
						// 若插入Node后,链表的size>TREEIFY_THRESHOLD(8)则将链表转换为红黑树
						// 在treeifyBin方法中可以看出若数组元素小于MIN_TREEIFY_CAPACITY(64)
						// 则不会转换为红黑树,只会触发resize()
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
					// 找到该key
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
				// 调用回调函数,HashMap中并没有实现具体行为,设计给其子类使用
                afterNodeAccess(e);
                return oldValue;
            }
        }
		// fail-fast标识
        ++modCount;
		// 若put元素后大于扩容阈值,则执行resize()
        if (++size > threshold)
            resize();
		// 插入完成后的回调，HashMap中并没有实现具体行为,设计给其子类使用
        afterNodeInsertion(evict);
        return null;
    }
```

&emsp;&emsp;从代码中可以看出hash值相等并不是判断两个key相同的唯一条件,两个key相同还必须满足equals结果为true.这里总结一下`hashCode`的关键点:

- HashCode的存在主要是为了查找的快捷性,HashCode是用来在散列存储结构中确定对象的存储地址
- 如果两个对象equals相等,那么这两个对象的HashCode一定也相同
- 如果对象的equals方法被重写,那么对象的HashCode方法也尽量重写
- 如果两个对象的HashCode相同,不代表两个对象就相同,只能说明这两个对象在散列存储结构中,存放于同一个位置

#### 扩容

&emsp;&emsp;根据上文分析,触发`resize()`为:1.`size` > `threshold` 2.`treeifyBin`时判断`size` < `MIN_TREEIFY_CAPACITY`(64).

```java
	final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 原容量>=2^30,不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量加倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // oldCap<=0,初始化HashMap对象时,table并未初始化(延迟初始化)
	// 设置了initialCapacity,此时使用threshold(>=initialCapacity的2的N次方幂)作为table初始化长度
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
	// 未设置initialCapacity,使用默认参数初始化
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
	// 设置新的threshold(仅当oldCap<=0切设置了initialCapacity会进入这个判断)
	// 感觉这个判断可以写在上段else if (oldThr > 0)的判断内,为何单独抽取出来,有意为之?
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 分配新table空间
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 若原来的表!=null,表名table已经被初始化过,需要将Node散列到新的位置
    if (oldTab != null) {
        // 遍历所有的bins
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 旧表中赋null,便于GC
                // 该bin中只有一个节点,直接散列到新的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 该bin中是一颗红黑树,通过红黑树的split方法处理
				// 将TreeNode拆解为2个TreeNode,原因与后面链表拆分原因一样.
				// 若拆分后TreeNode节点数量<=UNTREEIFY_THRESHOLD(6),即将其转换为链表
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 该bin中单项链表,重新散列,但要保证Entry原来的顺序
                // 因为容量加倍了,散列时使用的位数扩展了一位
                // 该链表中Node散列后仅可能有两个位置,通过新扩展位为0或1区分
                else { // preserve order
                    // 原链表分成两个链表,一高一低,通过新扩展的位来确定
					// 头节点用以最终放入table中,尾节点用于插入Node时的遍历
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 新扩展位为0(oldCap为2^k),低链表,索引不变
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 新扩展位为1,高链表,索引为原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // 低链表在数组中仍在原来的位置
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 高链表的位置相对于低链表的偏移为oldCap
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

&emsp;&emsp;这里有必要重述一下扩容后,由于容器长度变为原来的两倍,普通的做法为需要重新计算元素hash值找到新的索引位置.java8之前确实也是这么做的,但是java8对这里做了改进,可以发现规律:设n为数组长度,因为扩容后n变为原来2倍,那么n-1的mask范围在高位多1bit(红色),因此新的index就会发生这样的变化:

![HashMap 1.8 resize图示](http://okzr61x6y.bkt.clouddn.com/HashMap%201.8%20resize%E5%9B%BE%E7%A4%BA.png)

所以这里只需要判断`key.hash & oldCap`的结果为0则**索引不变**,结果为1则新索引为**原索引+oldCap**.

### LinkedHashMap

&emsp;&emsp;`LinkedHashMap`的出现主要是为了弥补`HashMap`的缺陷:`HashMap`是无序的,即遍历时的顺序一般不是添加元素的顺序.但是有些场景,我们希望拥有一个有序的`Map`.脑补一下设计这样一个有序的`Map`:首先它不能丢失`HashMap`通过hash值快速定位元素索引的特性(如果丢失这种特性,其实基本就和`LinkedList`差不多了),那么根据组合优于继承的设计原则,使用装饰器模式在有序`Map`中维护一个单例的`HashMap`成员变量.接下来就是增加有序功能,方法基本也别无选择,维护一个存储所有元素的双向链表.(有序性让我们想到了`ArrayList`和`LinkedList`,这里只需要遍历时有序,也就用不到`ArrayList`根据index快速随机访问的优势).

&emsp;&emsp;脑补如此,源码并非如此~`LinkedHashMap`继承了`HashMap`,并重写了`HashMap`的部分方法,增加额外空间开销维护一个双向链表来保证遍历时的有序性.

#### 有序性保证

&emsp;&emsp;为了保证有序性,在对`HashMap`进行增、删、改操作时需要同时维护双向链表.`LinkedHashMap`并没有重写任何put方法,而是重写了构建新节点的`newNode()`方法.

```java
	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
	
	// link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
		// 尾插法保证维护插入元素的顺序
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```

&emsp;&emsp;删除、修改元素都回调`LinkedHashMap`去操作双向链表,比较简单,就不贴代码了.

#### 访问有序性

&emsp;&emsp;分析到这里可能觉得`LinkedHashMap`设计的也挺好理解,没啥特别的思想.然而java集合类的设计者比我们思考的更远.因为`LinkedHashMap`设计的目的不仅仅只是为了满足插入有序性,它还支持访问有序性,通过成员变量`accessOrder`控制.

```java
	public LinkedHashMap() {
        super();
		// 默认情况为false,即只满足插入有序性
        accessOrder = false;
    }
	
	public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

&emsp;&emsp;特别说明一下,这里所指的访问仅为两种情况:

- 查询操作,即根据key查询value(`get(Object key)`、`getOrDefault(Object key, V defaultValue)`)
- 插入操作,`put(K key, V value)`时发生这样一种情况,即原key已经存在,此时put操作会修改原value值,这也算作是一种访问操作.

&emsp;&emsp;设置`accessOrder=true`时会将当前访问节点前后节点相连,并把该节点移到双向链表尾部,让其成为新的尾节点.(详见`void afterNodeAccess(Node<K,V> e)`方法).这么做的意义就是为了保证头节点是最不常访问的节点.此处想到了什么?对,就是`LRU cache`.(最近最少使用缓存,即当缓存满了,会优先淘汰那些最近最不常访问的缓存数据).

&emsp;&emsp;为了能够满足`LRU cache`的特性,这个伏笔在`HashMap`的put操作时就埋下了(详情请看`HashMap`代码分析中的`putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)`方法).其中参数`evict`中文翻译过来就是剔除的意思,顾名思义,插入一个元素,为了保持容器总容量不变必然需要剔除其中一个元素.对应到`LRU cache`中的操作就是:缓存总容量固定,当缓存已满时,此时再插入缓存数据必然伴随着删除一个最近最少使用缓存数据.

```java
	void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
		// 可以剔除节点,且头节点不为null,且满足removeEldestEntry(first)=true
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
	
	// LinkedHashMap中该方法返回false,即表示默认不开启删除头节点操作
	// 若想将其改造为容量固定的LRU cache,只需重写该方法 size() > CACHE_MAXIMUM_CAPACITY(缓存最大容量);
	protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

### end

&emsp;&emsp;java集合类的解读文章基本已经烂大街了,所以也很难写出干货,只能尽量写出深度,传达其设计思想.其实可以看出`HashMap`和`LinkedHashMap`的关联性非常强,甚至说`HashMap`很多细节就是为`LinkdedHashMap`而设计.在我看来这种父类针对子类的过度设计似乎不是那么优雅,但是也无可奈何.我们在代码设计时也常遇到这种痛点,子类扩展父类功能时往往只需修改父类一点点代码,子类的代码便会好写很多.但是这似乎又违反设计模式中对扩展开放、对修改关闭的设计原则.

&emsp;&emsp;无奈归无奈,小伙伴们在代码设计时可以多思考代码扩展性,脑补各种可能改动的点,然后加上一些钩子方法供子类扩展,但是也不要太过度设计.毕竟软件开发中常常是需求功能稳定后才开始重构代码.
