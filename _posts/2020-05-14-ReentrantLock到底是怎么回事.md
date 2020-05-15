---
layout:     post                    # 使用的布局（不需要改）
title:      ReentrantLock是怎么回事        # 标题 
subtitle:   ReentrantLock  #副标题
date:       2020-05-14          # 时间
author:     MistRay                      # 作者
# header-img: img/post-bg-7.jpg 这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - aqs
    - 锁
---
## 前言
>A reentrant mutual exclusion {@link Lock} with the same basic 
behavior and semantics as the implicit monitor lock accessed using
{@code synchronized} methods and statements, but with extended
capabilities.

以上摘自ReentrantLock源码第一句话，意思是ReentrantLock和synchronized的行为
和语义基本相同，但是扩展了部分能力。

ReentrantLock重入锁，是实现Lock接口的一个类，也是在实际编程中使用频率很高的一个锁，支持重入性，
表示能够对共享资源能够重复加锁，即当前线程获取该锁再次获取不会被阻塞。其中最重要的两个特性是:

1. 可重入性
2. 支持公平锁和非公平锁

### 类图
![post_2020_05_14_01ReentrantLock类图.jpg](/img/post_img/post_2020_05_14_01ReentrantLock类图.jpg)

ReentrantLock内部使用了大量的cas操作来保证无锁并发。同时使用了广为人知的AQS框架。

队列同步器AbstractQueuedSynchronizer，简称AQS，
是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，
通过内置的FIFO队列来完成资源获取线程的排队工作，并发包的作者（Doug Lea）期望它能够成为实现大部分同步需求的基础。
                                  

## 源码解析

话不多说，`Talk is cheap. Show me the code.`,让我们直接进入源码解析环节。

### 引子

下面让我们来看一下ReentrantLock的简单使用
```java
    public static void main(String[] args) {
        // 创建一个锁对象
        ReentrantLock lock = new ReentrantLock();
        // 加锁
        lock.lock();
        // 解锁
        lock.unlock();
    }
```
### 构造函数
首先让我们看下ReentrantLock的构造函数.

```java
    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        // 无参构造默认使用了非公平锁
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        // true为公平锁,false为非公平锁
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### 非公平锁实现
#### 加锁
让我们从ReentrantLock的`lock()`函数开始分析.
```java
    /**
     * Acquires the lock.
     *
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        // sync即在构造时创建的不同实现
        sync.lock();
    }
        /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            // 插队先尝试获取一下锁,调用aqs的cas函数,将state从0更新为1
            if (compareAndSetState(0, 1))
                // 如果成功,则直接将锁和当前线程绑定,即加锁成功
                setExclusiveOwnerThread(Thread.currentThread());
            else
                // 如果失败,那就说明了锁已经被其他线程占有
                acquire(1);
        }
        // aqs本身不提供实现,所以这里复写了该函数
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

acquire(int arg)函数是NonfairSync的抽象父类Sync的父类AbstractQueuedSynchronizer的逻辑,内容如下.

```java
    public final void acquire(int arg) {
        // 当获取同步状态失败，并且自旋不断获取同步状态
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 阻塞当前线程
            selfInterrupt();
    }
    // aqs本身不提供实现,所以NonfairSync复写了该函数
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```
**nonfairTryAcquire**  

这里我们跟到了抽象类Sync的nonfairTryAcquire(int acquires)函数中.
```java
        final boolean nonfairTryAcquire(int acquires) {
            // 拿到了当前线程
            final Thread current = Thread.currentThread();
            // 获取了当前锁的状态
            int c = getState();
            // 如果是0,说明没有别的线程持有锁
            if (c == 0) {
                // 直接通过cas加锁
                if (compareAndSetState(0, acquires)) {
                    // 将当前线程与锁绑定
                    setExclusiveOwnerThread(current);
                    // 加锁成功
                    return true;
                }
            }
            // 说明当前锁状态为已上锁,判断加锁线程是否为当前线程
            else if (current == getExclusiveOwnerThread()) {
                // 如果是,则将锁的重入次数+1
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                // 加锁成功
                return true;
            }
            return false;
        }
```

**addWaiter**  

继续看aqs内acquire函数的第二个内容`addWaiter`.
同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Nodeexpect,Nodeupdate)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。
```java
    // 入参mode上面写死了一个Node.EXCLUSIVE
    private Node addWaiter(Node mode) {
        // 创建了代表了当前线程的节点
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 如果链表不为空的话,尝试快速添加
        if (pred != null) {
            // 讲当前线程节点放到链表的尾部
            node.prev = pred;
            // 传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 否则，通过死循环来保证节点的正确添加
        enq(node);
        return node;
    }
    

    private Node enq(final Node node) {
        //死循环
        for (;;) {
            // 获取到尾节点
            Node t = tail;
            // t==null说明链表是空链表
            if (t == null) { // Must initialize
                // 将当前线程节点设置为头节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 将尾节点设置为当前线程节点的上一个节点
                node.prev = t;
                // 递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
在enq(final Node node)方法中，同步器通过“死循环”来保证节点的正确添加，在“死循环”中只有通过CAS将节点设置成为尾节点之后，当前线程才能从该方法返回，否则，当前线程不断地尝试设置。可以看出，enq(final Node node)方法将并发添加节点的请求通过CAS变得“串行化”了。

**串行化的优点**

如果通过加锁同步的方式添加节点，线程必须获取锁后才能添加尾节点，那么必然会导致其他线程等待加锁而阻塞，获取锁的线程释放锁后阻塞的线程又会被唤醒，而线程的阻塞和唤醒需要依赖于系统内核完成，因此程序的执行需要从用户态切换到核心态，而这样的切换是非常耗时的操作。如果我们通过”循环CAS“来添加节点的话，所有线程都不会被阻塞，而是不断失败重试，线程不需要进行锁同步，不仅消除了线程阻塞唤醒的开销而且消除了加锁解锁的时间开销。但是循环CAS也有其缺点，循环CAS通过不断尝试来添加节点，如果说CAS操作失败那么将会占用处理器资源。

**acquireQueued**

节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说是线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中。


```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 死循环
            for (;;) {
                final Node p = node.predecessor();
                //前驱节点是首节点且获取到了同步状态
                if (p == head && tryAcquire(arg)) {
                    // 设置首节点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //获取同步状态失败后判断是否需要阻塞或中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 获取前驱节点的等待状态
        int ws = pred.waitStatus;
        //SIGNAL状态：前驱节点释放同步状态或者被取消，将会通知后继节点。因此，可以放心的阻塞当前线程，返回true。
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        //前驱节点被取消了，跳过前驱节点并重试
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //独占模式下，一般情况下这里指前驱节点等待状态为SIGNAL
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
             //设置当前节点等待状态为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    
    private final boolean parkAndCheckInterrupt() {
        // 阻塞当前线程
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
可以看到节点和节点之间在循环检查的过程中基本不相互通信，而是简单地判断自己的前驱是否为头节点，这样就使得节点的释放规则符合FIFO。并且也便于对过早通知的处理（过早通知是指：前驱节点不是头节点的线程由于中断而被唤醒）。
至此加锁的逻辑久结束了。

#### 解锁
解锁的逻辑，我们从ReentrantLock的unlock函数向下跟进
```java
    public void unlock() {
        sync.release(1);
    }
```
短短的只有一行，这个release函数是aqs的函数，且并没有被其子类重写。

当前线程获取同步状态并执行了相应逻辑之后，就需要释放同步状态，使得后续节点能够继续获取同步状态。通过调用同步器的release(int arg)方法可以释放同步状态，该方法在释放了同步状态之后，会"唤醒"其后继节点（进而使后继节点重新尝试获取同步状态）。

```java
    public final boolean release(int arg) {
        // 释放同步状态
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 唤醒后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    // tryRelease函数被抽象类Sync复写，所以公平锁和非公平锁的实现是相同的
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```

**tryRelease**

tryRelease函数被ReentrantLock的内部抽象类Sync复写，所以公平锁和非公平锁的实现是相同的，下面让我们进入tryRelease函数，看看到底干了些什么。
```java
        protected final boolean tryRelease(int releases) {
            // 获取当前锁的重入次数并减1
            int c = getState() - releases;
            // 如果当前线程不是占有锁的线程，那么抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // 如果锁重入次数是0的话，说明解锁成功，将free置为true
            if (c == 0) {
                free = true;
                // 将锁绑定线程置为null
                setExclusiveOwnerThread(null);
            }
            // 由于解锁的逻辑发生在加锁成功之后，所以不存在争抢，直接设置即可
            setState(c);
            return free;
        }
```

**unparkSuccessor**

```java
    private void unparkSuccessor(Node node) {
        // 获取当前节点的等待状态
        int ws = node.waitStatus;
        if (ws < 0)
            // 更新等待状态
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        // 找到第一个没有被取消的后继节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            // 唤醒后继线程，里面调用了Unsafe的unpark函数（jni函数）
            LockSupport.unpark(s.thread);
    }
```

在获取同步状态时，同步器维护了一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋。移出队列（停止自旋）的条件是**前驱节点为头节点且成功获取了同步状态**。在释放同步状态时，同步器调用tryRelease（int arg）方法释放同步状态，然后唤醒头节点的后继节点。

## 总结
1. ReentrantLock在加锁时使用了cas操作来保证当前线程入队的正确性.
2. 在等待队列中自旋的节点只关心自己前驱节点,前驱节点为头节点且成功获取了同步状态.
3. ReentrantLock的可重入特性是通过state参数来保证的
4. ReentrantLock公平锁和非公平锁的实现十分相似,非公平锁在加锁的时候会先去尝试加锁,失败后尝试入队,而公平锁直接尝试入队,
在争抢锁的逻辑中,公平锁多了一个hasQueuedPredecessors函数作为校验,只有当前节点为队列头或队列为空时才可通过校验

## Reference
[队列同步器（AQS）详解](https://blog.csdn.net/sunxianghuang/article/details/52287968)  
[ReentrantLock原理](https://blog.csdn.net/fuyuwei2015/article/details/83719444)  
[AQS实践（一）：ReentrantLock概述](https://blog.csdn.net/qq_32573109/article/details/103304261)  
[Oracle官方文档](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html)  
[《The java.util.concurrent Synchronizer Framework》 JUC同步器框架（AQS框架）原文翻译](https://www.cnblogs.com/dennyzhangdd/p/7218510.html)  
[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)  
## 转载

本文遵循 [CC 4.0 by-sa](https://creativecommons.org/licenses/by-sa/4.0/) 版权协议,转载请附上原文出处链接和本声明。