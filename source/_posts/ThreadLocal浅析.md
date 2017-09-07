---
title: ThreadLocal浅析
categories: 技术
tags: [java,多线程]
---

## start

&emsp;&emsp;最近看java8函数性编程,看了下ThreadLocal类的源码,顿时豁然开朗,以前太笨了,也懒得看源码,反正只是知道这个类每个线程维护一个线程安全的变量,但是并未解决线程共享的问题.

- - -

<!--more-->

- - -

## content

### 理解

&emsp;&emsp;先说说我没看源码,也没看其他技术类文章前对ThreadLocal类的理解吧,我知道这个类是线程本地变量,做不到多线程的变量共享,所以嘛,我猜想是ThreadLocal是以Map实现,key存的是Thread,value存的是值,当调用get()方法时,会先获取currentThread,然后根据该key获取value.

&emsp;&emsp;看了源码之后,感觉自己还是单纯了啊.不废话,上代码:

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
        /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

&emsp;&emsp;可以清楚的看到存储信息确实是个map(ThreadLocalMap),key竟然是ThreadLocal本身,这确实出乎我的意料.在这个方法内部我们看到,首先通过getMap(Thread t)方法获取一个和当前线程相关的ThreadLocalMap,然后将变量的值设置到这个ThreadLocalMap对象中,当然如果获取到的ThreadLocalMap对象为空,就通过createMap方法创建.

&emsp;&emsp;线程隔离的秘密,就在于ThreadLocalMap这个类.ThreadLocalMap是ThreadLocal类的一个静态内部类,它实现了键值对的设置和获取（对比Map对象来理解）,每个线程中都有一个独立的ThreadLocalMap副本,它所存储的值,只能被当前线程读取和修改.ThreadLocal类通过操作每一个线程特有的ThreadLocalMap副本,从而实现了变量访问在不同线程中的隔离.因为每个线程的变量都是自己特有的,完全不会有并发错误.还有一点就是,ThreadLocalMap存储的键值对中的键是this对象指向的ThreadLocal对象,而值就是你所设置的对象了.

### ThreadLocalMap

```java
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

&emsp;&emsp;可以看出ThreadLocalMap里的Entry竟然用的是弱引用,具体map实现比较复杂,我一时没看明白,也就不贴代码,这个以后专门出一篇文章说说map的实现,毕竟java8在这方面改进了许多.

&emsp;&emsp;说说弱引用吧,为什么ThreadLocalMap的用弱引用的Entry来存储呢?WeakReference标志性的特点是：reference实例不会影响到被应用对象的GC回收行为(即只要对象被除WeakReference对象之外所有的对象解除引用后,该对象便可以被GC回收),只不过在被对象回收之后,reference实例想获得被应用的对象时程序会返回null.

&emsp;&emsp;被废弃了的ThreadLocal所绑定对象的引用,会在以下4情况被清理.如果此时外部没有绑定对象的引用,则该绑定对象就能被回收了:

- 1 Thread结束时.

- 2 当Thread的ThreadLocalMap的threshold超过最大值时.

- 3 向Thread的ThreadLocalMap中存放一个ThreadLocal,hash算法没有命中既有Entry,而需要新建一个Entry时.

- 4 手工通过ThreadLocal的remove()方法或set(null).

### 内存泄漏

&emsp;&emsp;如果我们粗暴的把ThreadLocal设置null,而不调用remove()方法或set(null),那么就可能造成ThreadLocal绑定的对象长期也能被回收，因而产出内存泄露。

&emsp;&emsp;ThreadLocalMap使用ThreadLocal的弱引用作为key,如果一个ThreadLocal没有外部强引用来引用它,那么系统GC的时候,这个ThreadLocal势必会被回收,这样一来,ThreadLocalMap中就会出现key为null的Entry,就没有办法访问这些key为null的Entry的value,如果当前线程再迟迟不结束的话,这些key为null的Entry的value就会一直存在一条强引用链:Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收,造成内存泄漏.

&emsp;&emsp;下面我们分两种情况讨论:
- key 使用强引用:引用的ThreadLocal的对象被回收了,但是ThreadLocalMap还持有ThreadLocal的强引用,如果没有手动删除,ThreadLocal不会被回收,导致Entry内存泄漏.
- key 使用弱引用:引用的ThreadLocal的对象被回收了,由于ThreadLocalMap持有ThreadLocal的弱引用,即使没有手动删除,ThreadLocal也会被回收.value在下一次ThreadLocalMap调用set,get,remove的时候会被清除.

比较两种情况,我们可以发现:由于ThreadLocalMap的生命周期跟Thread一样长,如果都没有手动删除对应key,都会导致内存泄漏,但是使用弱引用可以多一层保障:弱引用ThreadLocal不会内存泄漏,对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除.
因此,ThreadLocal内存泄漏的根源是:由于ThreadLocalMap的生命周期跟Thread一样长,如果没有手动删除对应key就会导致内存泄漏,而不是因为弱引用.

### 最佳实践

综合上面的分析,我们可以理解ThreadLocal内存泄漏的前因后果,那么怎么避免内存泄漏呢?
每次使用完ThreadLocal,都调用它的remove()方法,清除数据.
在使用线程池的情况下,没有及时清理ThreadLocal,不仅是内存泄漏的问题,更严重的是可能导致业务逻辑出现问题.所以,使用ThreadLocal就跟加锁完要解锁一样,用完就清理.

### end

&emsp;&emsp;最近看了很多常用类的源码,发现都引入了函数性接口,ThreadLocal初始化变量也用到了:
```java
    public static final ThreadLocal<Integer> localInteger = ThreadLocal
        .withInitial(() -> new Integer(0));
```
&emsp;&emsp;这周主要任务就是熟练使用函数性编程代码风格>_<
