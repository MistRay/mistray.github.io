---
layout:     post                    # 使用的布局（不需要改）
title:      ThreadLocal真的会内存泄露么        # 标题 
subtitle:   ThreadLocal  #副标题
date:       2020-02-28          # 时间
author:     MistRay                      # 作者
# header-img: img/post-bg-7.jpg 这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - ThreadLocal
    - Thread
---
## 前言

标题所述的问题来源于我朋友的某次面试。我朋友具体是怎么回答的我现在已经忘记了，但这个问题勾起了我的兴趣，所以就有了这篇文章。

## ThreadLocal
> This class provides thread-local variables.  These variables differ from their normal counterparts in that each thread that accesses one (via its {@code get} or {@code set} method) has its own, independently initialized copy of the variable.

上面这段文字来自于ThreadLocal注释的开头。意思是，此类提供线程局部变量。这些变量与正常变量不同，因为每个访问一个线程（通过其{@code get}或{@code set}方法）的线程都有其自己的独立初始化的变量副本。 
简单点说，它是一个数据结构，有点像HashMap，可以保存"key : value"键值对，但是一个ThreadLocal只能保存一个，并且各个线程的数据互不干扰。

### get&set
现在让我们来看下get和set方法的源码
```java

    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        // 获取到了当前的线程
        Thread t = Thread.currentThread();
        // 从当前线程中获取到了ThreadLocalMap
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
        // 从当前线程中获取到ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
可以发现无论是get还是set方法，操作的都是一个叫ThreadLocalMap的东西。那就让我们来看下这个ThreadLocalMap到底是什么。

### ThreadLocalMap
ThreadLocalMap是ThreadLocal中的一个静态内部类，从名字就可以看出ThreadLocalMap是一个Map，但ThreadLocalMap并没有实现Map接口。

ThreadLocalMap中也是初始化一个大小16的Entry数组，Entry对象用来保存每一个key-value键值对，不过这里的key永远都是ThreadLocal对象。通过ThreadLocal对象的set方法，结果把ThreadLocal对象自己当做key，放进了ThreadLocalMap中。

#### ThreadLocalMap的构造函数
```java
     /**
      * Construct a new map initially containing (firstKey, firstValue).
      * ThreadLocalMaps are constructed lazily, so we only create
      * one when we have at least one entry to put in it.
      */
     ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
         // 和HashMap相同，内部是一个Entry数组,默认初始化长度为16
         table = new Entry[INITIAL_CAPACITY];
         // 算出ThreadLocal在数组里面的所在位置
         int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
         // 把K,V组成的Entry扔到数组里
         table[i] = new Entry(firstKey, firstValue);
         size = 1;
         // 设置扩容的阈值
         setThreshold(INITIAL_CAPACITY);
     }
     
     /**
      * Set the resize threshold to maintain at worst a 2/3 load factor.
      */
     private void setThreshold(int len) {
         threshold = len * 2 / 3;
     }

```
#### hash冲突
没有链表结构，那发生hash冲突了怎么办？  
先看看ThreadLocalMap中插入一个key-value的实现

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

每个ThreadLocal对象都有一个hash值threadLocalHashCode，每初始化一个ThreadLocal对象，hash值就增加一个固定的大小0x61c88647。

在插入过程中，根据ThreadLocal对象的hash值，定位到table中的位置i，过程如下：
1、如果当前位置是空的，那么正好，就初始化一个Entry对象放在位置i上；
2、不巧，位置i已经有Entry对象了，如果这个Entry对象的key正好是即将设置的key，那么重新设置Entry中的value；
3、很不巧，位置i的Entry对象，和即将设置的key没关系，那么只能找下一个空位置；

这样的话，在get的时候，也会根据ThreadLocal对象的hash值，定位到table中的位置，然后判断该位置Entry对象中的key是否和get的key一致，如果不一致，就判断下一个位置

可以发现，set和get如果冲突严重的话，效率很低，因为ThreadLocalMap是Thread的一个属性，所以即使在自己的代码中控制了设置的元素个数，但还是不能控制其它代码的行为。


## ThreadLocal会导致内存泄露？
空口无凭，这里我写了一个内存泄露的例子
```java
    public static void main(String[] args) {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        try {
            threadLocal.set("MistRay");
            // 消除ThreadLocal的强引用
            threadLocal = null;
            System.gc();
            Thread.sleep(20000);
            Thread thread = Thread.currentThread();
            System.out.println(thread);
        } catch (Exception e) {
            System.out.println(1111);
        } finally {
            threadLocal.remove();
        }
    }
```
![Thread内容](/img/post_img/post_2020_04_09_01Thread内容.jpg)

referent为null，可以看出table里的元素就是ThreadLocalMap中的Entry。

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
从Entry中可以看出一些端倪，Entry的父类是WeakReference，也就是弱引用，referent是WeakReference构造函数的参数。
而这里将ThreadLocal作为弱引用的对象，而弱引用的对象只要发生GC就会被回收。（以后会专门写一篇文章来介绍Java中的几种引用）

所以，在唯一的强引用被我主动消除后，这里的弱引用无法支撑ThreadLocal不被GC回收，而Entry的value，只要线程不结束，
就会有一个引用链即:Thread->ThreadLocalMap->Entry->value，所以如果不做处理，value就会一直留存在内存中。

想复现内存泄露必须满足以下两个条件：  
1. 线程的生命周期很长，且重复利用（例如线程池中的线程）
2. 没有调用ThreadLocal的remove函数

正确的使用姿势是这样的
```java
    ThreadLocal<String> threadLocal = new ThreadLocal();
    try {
        threadLocal.set("MistRay");
    } finally {
        threadLocal.remove();
    }
```

## 总结
ThreadLocal如果使用不当真的会内存泄露。但只要掌握了正确的姿势，就完全不用担心泄露啦~



## Reference
[Java面试必问，ThreadLocal终极篇](https://www.jianshu.com/p/377bb840802f)  
[深入分析 ThreadLocal](http://www.iocoder.cn/JUC/sike/ThreadLocal/)  
[ThreadLocal就是这么简单](https://juejin.im/post/5ac2eb52518825555e5e06ee)  
[ThreadLocal-面试必问深度解析](https://www.jianshu.com/p/98b68c97df9b)  
[ThreadLocal内存泄漏真因探究](https://www.jianshu.com/p/a1cd61fa22da)  
[Java中的四种引用类型（强、软、弱、虚）](https://www.jianshu.com/p/ca6cbc246d20)
## 转载

本文遵循 [CC 4.0 by-sa](https://creativecommons.org/licenses/by-sa/4.0/) 版权协议,转载请附上原文出处链接和本声明。