---
title: java并发之AQS框架解析
categories: 技术
tags: [java,并发]
---

## start

&emsp;&emsp;今天来介绍下java5引入的**AQS**(类名全称`AbstractQueuedSynchronizer`)框架.java5引入的很多并发类都是基于**AQS**框架构建,所以想弄清楚整个java的并发体系,很有必要深入研究一下这个框架.

- - -

<!--more-->

- - -

## content

### 概要

&emsp;&emsp;本文旨在解析**AQS**的设计思想,每一种同步器虽然作用不同(比如`ReentrantLock`、`Semaphore`、`Barrier`等),但是都涉及到一些共性的操作:

- 阻塞和非阻塞之间状态的转换
- 可选的超时设置,让调用者可以放弃等待
- 针对不同形式的独占或共享的释放和获取

&emsp;&emsp;框架的意义就是为了整合共性,兼容异性.**AQS**框架完美地通过一种并发模型去支持各种不同并发同步器.

### 锁

&emsp;&emsp;在正式介绍**AQS**框架之前,先介绍一些相关的锁的概念.

#### 自旋锁

- CLH锁:是一种基于链表的可扩展、高性能、公平的自旋锁,申请线程在前驱节点属性上自旋,它不断轮询前驱的状态,如果发现前驱释放了锁就结束自旋.能确保无饥饿性,提供先来先服务的公平性.
- MCS锁:是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地属性上自旋,直接前驱负责通知其结束自旋,从而极大地减少了不必要的处理器缓存同步的次数,降低了总线和内存的开销.

&emsp;&emsp;这两种锁有点类似观察者模式中的推、拉模式,**CLH锁**主动去拉前驱节点的加锁状态进行自旋操作,而**MCS锁**则是主动通知后继节点的加锁状态变更.AQS框架选用了CLH锁(准确的来说是CLH锁的变体),主要因为:

- CLH锁可以更容易去实现取消和超时功能,通过在节点中显式地维护前驱节点,如果一个节点的前驱节点取消了,这个节点就可以依次向前遍历去检查第一个未取消前驱节点的状态.
- CLH锁入队和出队操作快速、且非阻塞(即使在竞争条件下,一个线程总是会在竞争中不断的执行CAS操作),检测到其他线程是否等待的过程也很快(只需要检测头节点和尾节点是否一样),释放状态是分散的,避免了内存的竞争.

#### 公平性

&emsp;&emsp;公平锁和非公平锁的队列都基于锁内部维护的一个双向链表,表结点Node的值就是每一个请求当前锁的线程.公平锁的作用就是严格按照线程启动的顺序来执行的,不允许其他线程插队执行,而非公平锁是允许插队的.公平锁每次都是依次从队首取Node占有锁.非公平锁每个Node都可以去竞争锁.在性能层面来说,非公平锁的性能更好,一个线程多次获取到锁,并没有按照队列顺序获取,而公平锁加了必须为队首Node的限制条件,这样就导致线程上下文切换的次数更多一些.(竞争不到锁,必然会阻塞当前线程,进行上下文切换)

&emsp;&emsp;这里我以前的理解一直有偏差,所以需要再强调一下.我以前的对非公平性的理解是:当一个线程释放锁时,其他在等待队列里的线程都有机会去抢占锁.我的想法并没有问题,但是这种思路并不好代码实现(用随机数和队列长度随机定位到队列中的一个Node,让其抢占到锁,需要额外维护队列长度,性能实在很不理想).AQS框架中的非公平性指的是:**当一个线程释放锁时,等待队列中的队首Node会去和此时一个新的想要获取锁的线程(如果有的话)去竞争锁.**这个新的插队线程便是非公平性的体现.但是已经进去等待队列的线程依然是公平地按照队首Node优先竞争锁.

#### 独占性

&emsp;&emsp;AQS框架中的节点支持共享(SHARE)和独占(EXCLUSIVE)两种模式.共享模式指的是允许多个线程获取同一个锁而且可能获取成功,独占模式指的是一个锁如果被一个线程持有,其他线程必须等待.例如:多个线程读取一个文件可以采用共享模式,而当有一个线程在写文件时不会允许另一个线程写这个文件,此时为独占模式.独占模式下,线程释放锁时会唤醒其后继节点.而共享模式下,其唤醒具有传播性,即可依次唤醒后继多个节点.

### 设计

&emsp;&emsp;AQS作为实现阻塞锁以及相关同步器的框架,总体设计上很直观:
```java
	// 申请锁
	// 1.不断地尝试判断同步状态并竞争锁
	// 2.竞争失败且不在等待队列中则入队列,并阻塞当前线程
	// 3.竞争成功且在等待队列中则出队列
	while (synchronization state does not allow acquire) {
    	enqueue current thread if not already queued;
    	possibly block current thread;
	}
	dequeue current thread if it was queued;
	
	// 释放锁
	// 1.更新同步状态
	// 2.如果允许一个或多个线程竞争锁则唤醒一个或多个线程
	update synchronization state;
	if (state may permit a blocked thread to acquire) {
	    unblock one or more queued threads;
	}
```

&emsp;&emsp;AQS框架是通过一个双向的FIFO同步队列来完成同步状态的管理,当有线程获取锁失败后,就被添加到队列末尾.队列中的元素Node就是保存着线程引用和线程状态的容器,每个线程对同步器的访问,都可以看做是队列中的一个节点.

```java
     //      +------+  prev +-----+       +-----+
     // head |      | <---- |     | <---- |     |  tail
     //      +------+       +-----+       +-----+
	 
	static final class Node {
	 
		// 该等待同步的节点处于共享模式
		static final Node SHARED = new Node();
		
		// 该等待同步的节点处于独占模式
        static final Node EXCLUSIVE = null;

		// 节点状态值,值为0:初始化,无状态
		// 值为1:表示当前节点被取消
        static final int CANCELLED =  1;
		
		// 值为-1:表示当前节点的的后继节点将要或者已经被阻塞,在当前节点释放的时候需要unpark后继节点
        static final int SIGNAL    = -1;
		
		// 值为-2:表示当前节点在等待condition,即在condition队列中
        static final int CONDITION = -2;
		
		// 值为-3:表示releaseShared需要被传播给后续节点(仅在共享模式下使用)
        static final int PROPAGATE = -3;

		// 当前节点的状态,对应上述节点状态值常量
        volatile int waitStatus;
	
		// 前驱节点
        volatile Node prev;

		// 后继节点
        volatile Node next;

		// 等待锁的线程
        volatile Thread thread;

		// 存储condition队列中的后继节点
        Node nextWaiter;
	}
```

&emsp;&emsp;为了维护这个Node链表队列,AQS需要持有Node头节点、Node尾节点以及同步状态值.

```java
	// 头节点
    private transient volatile Node head;

	// 尾节点
    private transient volatile Node tail;

	// 同步状态,0代表没有被占用,大于0代表有线程持有当前锁(可重入锁)
    private volatile int state;
```

&emsp;&emsp;AQS框架通过`LockSupport.park(thread);/LockSupport.unpark(thread);`阻塞/唤醒线程.这里介绍下其特点:

- LockSuppor只有1个许可可供使用,如果对一个线程进行多次unpark,只进行一次park,结果是许可处于可用状态,反之会一直阻塞下去
- LockSupport是不可重入的
- LockSupport会响应信号,但是不会抛出InterruptedException,也不会重置中断信号

```java
    @Test
    public void testPark() throws InterruptedException {
        Thread thread = new Thread(() -> {
            long start = System.currentTimeMillis();
            LockSupport.park();
            System.out.println("cost:" + (System.currentTimeMillis() - start));
        });
        thread.start();
        Thread.sleep(2000L);
        thread.interrupt();
        System.out.println(thread.isInterrupted()); // true,未重置中断信号
    }

    @Test
    public void testSleep() throws InterruptedException {
        Thread thread = new Thread(() -> {
            long start = System.currentTimeMillis();
            try {
                Thread.sleep(10000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("cost:" + (System.currentTimeMillis() - start));
        });
        thread.start();
        Thread.sleep(2000L);
        thread.interrupt();
        System.out.println(thread.isInterrupted()); // false,重置了中断信号
    }
```

&emsp;&emsp;一个基于AQS框架的同步器所执行的基本操作为一些不同形式的获取和释放.获取操作是状态依赖的操作.AQS中的`state`变量正是代表这一状态信息.例如:`ReentrantLock`用它来表示拥有它的线程已经请求了多少次锁,`Semaphore`用它来表现剩余的许可数,`FutureTask`用它来表现任务的状态(尚未开始、运行、完成和取消).当不满足相关状态,获取操作便会阻塞线程.而唤醒阻塞线程的关键便是释放操作,释放操作不是一个可阻塞的操作,它允许线程在请求执行前阻塞.

&emsp;&emsp;上述提到的获取操作可能是独占的(如`ReentrantLock`),也可能是非独占的(如`Semaphore`、`CountDownLatch`),这取决于不同同步器的功能.AQS框架提供了几个`protected`方法,需要子类同步器自行实现,支持独占获取的同步器应该实现`tryAcquire`、`tryRelease`和`isHeldExclusively`这几个方法,而支持共享获取的同步器应该实现`tryAcquireShared`、`tryReleaseShared`这几个方法.

```java
	// 独占获取
    public final void acquire(int arg) {
		// 1.尝试获取(tryAcquire由子类实现)
		// 2.获取失败则入队列(此时会不断阻塞)
		// 3.设置中断
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    private Node addWaiter(Node mode) {
        Node node = new Node(mode);

        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
				// 已初始化
                node.setPrevRelaxed(oldTail);
				// CAS将Node添加到队尾
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
				// 延迟初始化head/tail节点
                initializeSyncQueue();
            }
        }
    }

    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
				// 获取前驱节点
                final Node p = node.predecessor();
				// p == head 说明当前节点虽然进到了阻塞队列,但是是阻塞队列的第一个节点,因为它的前驱是head
				// 阻塞队列不包含head节点,head一般指的是占有锁的线程,head后面的才称为阻塞队列
				// 尝试获取锁(如果是之前是初始化head,此时head==tail==new Node(),没有真正拥有锁的线程,可以获取锁)
                if (p == head && tryAcquire(arg)) {
					// 获取锁则将当前节点设置为头节点
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
				// 为竞争到锁,判断是否需要挂起线程
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }
	
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
		// 前驱节点的waitStatus == -1,说明前驱节点正在持有锁,其后继节点需要阻塞
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
			// 前驱节点可能已经等待超时,此时需要找到一个正在等待锁的前驱节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
			// 新入队的节点waitStatus == 0,将其设为-1,虽然此次返回false,但会在下次循环中进入第一个条件,返回true
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        return false;
    }

    private final boolean parkAndCheckInterrupt() {
		// 挂起当前线程
        LockSupport.park(this);
        return Thread.interrupted();
    }
	
	// 独占释放
    public final boolean release(int arg) {
		// 1.尝试释放(tryRelease由子类实现)
		// 2.唤醒后继节点
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    private void unparkSuccessor(Node node) {
	
        int ws = node.waitStatus;
		// 修改头节点状态为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
		// 从尾节点开始往前找node后第一个未取消节点
		// 这里从尾节点往前遍历主要是因为addWaiter方法中添加尾节点并不是一个原子操作,CAS成功但未必能取到新的尾节点
		// if (compareAndSetTail(oldTail, node)) {
        //   	oldTail.next = node;
        // 		return node;
        // }
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
			// 唤醒后继节点
            LockSupport.unpark(s.thread);
    }

	// 共享获取
	public final void acquireShared(int arg) {
		// 尝试获取(tryAcquireShared由子类实现)
		// 返回<0说明资源不够,需要进入等待队列
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
				// 获取前驱节点
                final Node p = node.predecessor();
                if (p == head) {
					// 尝试获取,资源若>=0,传播唤醒后继节点
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
		// 设置node为head节点
        setHead(node);
		// 唤醒当前node之后的节点
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
	
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
			// h == null说明阻塞队列为空,h == tail说明刚刚初始化阻塞队列,这两种情况无需唤醒后继节点
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
					// 唤醒后继节点
                    unparkSuccessor(h);
                }
				// CAS失败说明此时恰好有后继节点将head的waitstatus设为-1
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
			// head变成唤醒的线程,继续循环
            if (h == head)                   // loop if head changed
                break;
        }
    }
	
	// 共享释放
    public final boolean releaseShared(int arg) {
		// 尝试释放(tryReleaseShared由子类实现)
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

&emsp;&emsp;下面会以`ReentrantLock`和`CountDownLatch`为例分别介绍AQS的独占和非独占两种形式.

### ReentrantLock

&emsp;&emsp;`ReentrantLock`为AQS的独占形式,支持公平锁和非公平锁,这里所谓的非公平为释放锁时,需要竞争锁的新线程可以插队与等待队列队首线程进行竞争获取锁.

```java
	
    static final class FairSync extends Sync {
	
        @ReservedStackAccess
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
			// status == 0说明此时无线程持有锁
            if (c == 0) {
				// 非公平锁只取消!hasQueuedPredecessors()判断条件,所以不再贴NonfairSync代码
				// 即在非公平锁下,竞争锁的线程无需是等待队列的队首节点
				// CAS更改状态
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
					// 设置独占锁线程为当前线程	
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
			// 重入锁支持
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
		// h == t 说明刚初始化等待队列,此时没有持有锁的前驱节点
		// 当前节点为队首节点(无前驱节点)
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

    abstract static class Sync extends AbstractQueuedSynchronizer {
		
        protected final boolean tryRelease(int releases) {
			// 在可重入的情况下,可能c > 0
            int c = getState() - releases;
			// 当前线程为持有锁的线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
				// c == 0时,说明重入锁已完全释放
                free = true;
				// 设置持有锁线程为null
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

### CountDownLatch

&emsp;&emsp;`CountDownLatch`为AQS的非独占形式.这里解释一下:假设我们有N(N > 0)个任务,那么我们会用 N来初始化一个`CountDownLatch`,然后将这个`latch`的引用传递到各个线程中,在每个线程完成了任务后,调用 `latch.countDown()`代表完成了一个任务.调用`latch.await()`的方法的线程会阻塞,直到所有的任务完成.

&emsp;&emsp;`latch.await()`的方法的线程会阻塞,可以分析出这肯定是一个获取操作(acquireShared).`latch.countDown()`代表完成了一个任务,即为一个释放操作(releaseShared).

```java
    private static final class Sync extends AbstractQueuedSynchronizer {

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

		// 初始化是时state > 0(代表初始化n个任务)
		// state == 0 说明所有任务已经完成,此时需要唤醒所有在等待队列中的已阻塞线程
		// state < 0 则需要将当前线程入等待队列
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
				// state == 0 说明所有任务已经完成,此时不能继续完成任务
                if (c == 0)
                    return false;
				// CAS设置state == c-1 说明完成了一个任务
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    public void await() throws InterruptedException {
		// 阻塞直至state == 0
        sync.acquireSharedInterruptibly(1);
    }

    public void countDown() {
		// state = c-1 完成一个任务
        sync.releaseShared(1);
    }
```

### end

&emsp;&emsp;半懂不懂地写完这篇文章,虽然感觉自己理解的还是不到位,还是惊叹源码作者的天才设计.一个优秀框架就是致力于做到包容所有相关性,兼容所有无关性.这也是我神往的编程进阶.

