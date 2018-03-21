---
title: java集合启示录(一)之ArrayList、LinkedList、CopyOnWriteArrayList
categories: 技术
tags: [java,集合,数据结构]
---

## start

&emsp;&emsp;开个系列坑打算细致研究下java集合类(基于java8版本),虽然以前或多或少看过部分集合类源码,但是一般都是因为使用过程中遇到问题或者单纯地为了面试.这次开坑的目的,不带有那么强的目的性,只为研究学习.当然我所谓地学习,并不只是分析学习这些java集合类的设计原理细节,更重要地是体会设计背后的思想.细节可能会时间地流逝而渐渐淡忘,但是思想会慢慢沉淀、愈加深刻.

- - -

<!--more-->

- - -

## content

### 概要

&emsp;&emsp;既然是系列坑的开篇,那么就来一起探究一下集合类中`List`接口的实现 类:`ArrayList`、`LinkedList`、`CopyOnWriteArrayList`的设计原理吧.

### ArrayList

#### 基本介绍

&emsp;&emsp;`ArrayList`就是一个底层以数组结构实现的集合,区别于数组初始化时必须设定固定的长度,`ArrayList`好比于动态数组,提供了自动扩容的功能.`ArrayList`内部使用`Object[] elementData`存放元素,当需要扩容时只需要重新创建一个容量为原数组容量1.5倍(`newCapacity = oldCapacity + (oldCapacity >> 1)`)的新数组,并且把原数组上的元素依次顺序拷贝到新数组上.

- 允许NULL,有序,允许重复数据,非线程安全
- 比较适合顺序添加、随机访问的场景
- 插入和删除时需要元素复制的过程,比较费性能

#### 源码细节

##### 扩容

```java
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 以前java版本为newCapacity = oldCapacity * 3 / 2;位运算符速度更快
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 以防newCapacity溢出
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

##### fail-fast

&emsp;&emsp;`fail-fast`机制是一种错误检测机制.当多个线程对同一个集合的进行操作并产生集合中元素结构变化时,就可能会产生`fail-fast`事件.例如:当某一个线程A通过`iterator`去遍历某集合的过程中,若该集合元素结构被其他线程所改变了,那么线程A访问集合时,就会抛出`ConcurrentModificationException`异常.这里所说的改变元素结构的操作指如添加元素、插入元素、删除元素这些会改变集合元素数量的操作,而修改元素不会改变集合元素数量,因此不会引发`fail-fast`机制.

&emsp;&emsp;`ArrayList`中通过成员变量`protected transient int modCount = 0;`作为在遍历集合过程中是否会触发`fail-fast`机制的标识.这里以`remove`和`replaceAll`方法作为说明:

```java
	public E remove(int index) {
        rangeCheck(index); //检查index是否越界
        modCount++; // 操作数+1
        E oldValue = elementData(index);
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
        return oldValue;
    }
    
    public void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final int expectedModCount = modCount; // 获取当前操作数作为期待值
        final int size = this.size;
        // 在遍历过程中,每次操作都判断当前操作数是否与期待值相同
        // 若不同,说明在遍历过程中发生了改变集合结构的操作,因此引发fail-fast
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            elementData[i] = operator.apply((E) elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

&emsp;&emsp;需要说明的是,这里的`modCount`并没有任何线程安全性的保证,所以在并发场景下,当多个操作使集合元素结构发生变化时,仅仅**可能**产生`fail-fast`事件,而不是一定会产生.这里我想法的是:如果给`modeCount`加锁必然会对性能有影响,并且`ArrayList`设计的初衷并不是用于高并发场景,高并发场景应考虑其他集合类,所以这里仅保证了单线程下的快速失败机制.

##### 序列化

&emsp;&emsp;查看源码发现`private transient Object[] elementData;`,这里修饰存放集合元素的数组用了`transient`关键字.用`transient`修饰`elementData`意味着不希望`elementData`数组被序列化.这是因为实际元素数量一定是小于等于数组长度的,很多情况下元素数小于数组长度,那么序列化整个数组显然没有必要.因此`ArrayList`中重写了`writeObject`方法：

```java
	private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject(); // 序列化非transient元素
        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size); // 写size(实际长度)
        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]); // 写实际元素
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException(); // 快速失败保证序列化过程中不改变集合元素结构
        }
    }
```

### LinkedList

#### 基本介绍

&emsp;&emsp;`LinkedList`是基于双向链表实现的.不同于数组按线性的顺序存储数据,是在每个节点里存放前后节点的指针,可以充分利用计算机内存空间,实现灵活的内存动态管理.但是链表失去了数组随机读取的优点,同时链表由于增加了节点的指针域,空间开销比较大.
![LinkedList结构图示](http://okzr61x6y.bkt.clouddn.com/LinkedList%E7%BB%93%E6%9E%84.png)

- 允许NULL,有序,允许重复数据,非线程安全
- 顺序添加元素效率比`ArrayList`较低,因为需要`new`一个节点对象,有性能开销
- 插入、删除元素一般效率比`ArrayList`较高(`ArrayList`慢在数组元素的批量拷贝以及扩容),但不绝对(`LinkedList`寻址较慢,若插入/删除位置靠后,此时`ArrayList`由于需要批量拷贝的元素较少,效率可能高于`LinkedList`).
- `LinkedList`实现了`Deque`(Double-ended Queue)接口,因此可以当做非线程安全的队列或栈使用.

#### 源码细节

&emsp;&emsp;查看`LinkedList`源码,对于集合的操作主要是依赖基于链表的一些基本操作,较为简单,这里不详细说明.`LinkedList`和`ArrayList`一样提供了`fail-fast`机制,也是一样的原理.下面主要说明`LinkedList`选用双向链表带来的优势.

##### 定位元素位置

&emsp;&emsp;`LinkedList`提供了根据`index`插入、删除元素.(`add(int index, E element)`、`remove(int index)`).如果使用单链表,那么只能从链表的头节点开始遍历,若需要操作的元素在链表较后的位置(即`index`接近于`size`).此时几乎就等于遍历了整个链表,寻址很慢.如果使用双向链表,便可从头节点或尾节点两个方向遍历,提高了链表后半部分元素的寻址效率.

```java
    Node<E> node(int index) {
        // assert isElementIndex(index);
		// 通过对比index和size/2,选择从头节点或尾节点遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

##### 双端队列

&emsp;&emsp;队列是一种只允许在一端删除而在另一端插入的数据结构(先进先出).而双端队列(`Deque`)是队列的一种拓展,它可以在队列的两端进行插入和删除元素,因此是一种具有队列和栈的性质的数据结构.

&emsp;&emsp;当`Deque`当做队列使用时(FIFO),出队列需要从头部删除元素(`removeFirst()`),入队列需要从尾部添加元素(`addLast(E e)`)即可.

&emsp;&emsp;当`Deque`当做栈使用时,这时入栈(`addFirst(E e)`),出栈(`removeFirst()`)的元素都在双端队列的头部进行.

### CopyOnWriteArrayList

#### 基本介绍

&emsp;&emsp;`CopyOnWriteArrayList`是一个写时复制的容器.当在容器中增、删、改元素的时候,不直接操作当前容器,而是先将当前容器进行**copy**一份,复制出一个新的容器,然后对新容器进行操作,最后将原容器的引用指向新的容器.

&emsp;&emsp;读取`CopyOnWriteArrayList`的时候读取的是该对象中的`Object[] array`,但是修改的时候,操作的是一个新的`Object[] array`,读和写操作的不是同一个对象,这是一种读写分离的思想,读取集合里面的数据未必是最新的数据,满足最终一致性.最终一致性对于分布式系统非常重要,它通过容忍一定时间的数据不一致,提升整个分布式系统的可用性与分区容错性,但是对于数据实时性要求非常高的场景并不适用,这种场景必须满足强一致性.

&emsp;&emsp;随着`CopyOnWriteArrayList`中元素的增加,**copy**代价将越来越昂贵,并且在内存中会短暂时间内同时存在两个大对象(下次GC时,旧的数组占用内存会被释放).因此,`CopyOnWriteArrayList`适用于读操作远多于修改操作并且数据量不是特别大的并发场景中.

#### 源码细节

##### 线程安全性

&emsp;&emsp;`CopyOnWriteArrayList`是线程安全的,对容器进行增、删、改元素操作时通过`ReentrantLock`加锁,保证同一时间只有一个线程对集合进行操作,并且对该对象中的`Object[] array`添加了`volatile`关键字修饰,保证`array`的引用发生变化后,能立刻被所有线程感知.(`happen-before`原则)关于并发锁,这里不展开讨论,以后有机会会专门开坑详细分析.

##### 对比Vector

&emsp;&emsp;说起线程安全的`List`,可能首先想到的是`Vector`.查看源码发现`Vector`是通过`synchronized`关键字修饰了几乎所有的操作方法(包括`get(int index)`、`size()`),以此保证线程安全性.所以其性能也相对较差.

&emsp;&emsp;`Vector`虽然通过加锁保证各个操作是线程安全的,但它只能够保证增、删、查、改的单个操作一定是原子的,不会被打断,如果组合起来用,并不能保证这个组合操作的线程安全性.所以,`Vector`也被称作有条件的线程安全,即所有单个的操作都是线程安全的,但是多个操作组成的操作序列却可能导致数据争用,因为在操作序列中控制流取决于前面操作的结果.因此,`Vector`和`ArrayList`一样,通过`fail-fast`机制检测失败.

&emsp;&emsp;`CopyOnWriteArrayList`并没有引入`fail-fast`机制(当然也不需要如此).每次遍历时使用旧的数据数组引用,在遍历过程并不会受到数组数据变化的影响,这里所说的遍历仅仅指迭代器遍历查询,在`CopyOnWriteArrayList`的迭代器中,`add(E e)`、`set(E e)`、`remove()`这些方法`throw new UnsupportedOperationException();`,即不被允许.

### end

&emsp;&emsp;`List`就介绍到这里,感觉对设计重点还是分析的比较细致的,至于一些api方法,感觉没必要展开讨论,毕竟贴大段分析操作数组、链表的代码实在很low.以后这个系列坑只写精髓,不写废话.
