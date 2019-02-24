---
title: ReentrantLock以及AQS实现原理
date: 2019-02-24 14:30:11
categories: 并发
toc: true
---

提到Java中的锁，我们可能第一时间想到的是Synchronized，今天来详细分析下ReentrantLock。ReentrantLock顾名思义可重入锁，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁还支持获取锁时的公平和非公平性选择。针对可重入锁的具体实现，需要从它的一个内部类Sync说起，Sync的父类是AbstractQueuedSynchronizer(简称AQS)。

### ReentrantLock锁的架构

ReentrantLock锁的架构相对来说比较简单，主要包括一个Sync内部抽象类以及Sync抽象类的两个实现类，结构示意图如下所示。

<img src="ReentrantLock.png">

AQS的父类AbstractOwnableSynchronizer(下面简称AOS)，它主要提供一个exclusiveOwnerThread属性，用于关联当前持有锁的线程，另外Sync的两个实现类从名字我们可以知道一个用于公平锁，一个用于非公平锁。下面是ReentrantLock的一些核心方法。

<img src="lock.png">

默认构造函数构造的是非公平锁，可通过一个布尔值来控制，具体的lock方法调用的是sync的lock，整体的结构看起来还是比较简单。FairSync和NonFairSync都继承自Sync，Sync继承自AQS，AQS是采用模板方法的设计模式，它作为基础并发组件，封装了一层核心并发操作(比如获取资源成功后封装成Node加入队列，对队列双向链表的处理)，接下来我们来详细看看AQS的工作原理。

### AQS同步队列

先来瞅瞅AQS中非常重要的几个字段以及相应的方法

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
    
    // 同步队列节点
    static final class Node {
        // ......
    }
    
    // 指向同步队列队头
    private transient volatile Node head;
    
    // 指向同步队列队尾
    private transient volatile Node tail;
    
    // 同步状态字段,0代表未被占用,1代表已被占用,核心字段
    private volatile int state;
    
    protected final int getState() {
        return state;
    }
    
    protected final void setState(int newState) {
        state = newState;
    }
    
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
    private Node enq(final Node node) {
        // ......
    }
    
    private Node addWaiter(Node mode) {
        // ......
    }
}
```

节点类，指向队头的head，指向队尾的tail，同步状态等字段构成了AQS，再来看看这个Node内部类，它是对每一个访问同步代码块的线程的封装，下面是同步队列的基本结构。

<img src="aqs.png">

- AQS内部有一个Node组成的同步队列，它是双向链表结构
- AQS内部通过state来控制同步状态，当执行到lock时，如果state=0，说明没有任何线程占有共享资源的锁，此时线程会获取到锁并把state设置为1，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待。

AQS内部又分为共享模式和独占模式，无论是共享模式还是独占模式，都维持着一个虚拟的同步队列，当请求锁的线程超过现有模式的限制时，会将线程包装成Node节点并将线程当前必要的信息存储到Node节点中，然后加入同步队列，这一系列的操作封装在AQS中，这也是它会作为一个基础组件的原因，Java中一系列的并发操作的基础都是该类。

### 基于AQS分析ReetrantLock独占锁模式的实现

ReetrantLock分为公平锁以及非公平锁，下面直接从非公平锁的角度来分析。

#### 初始化

```java
// 默认构造函数，构造非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

//根据传入参数创建锁类型
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}

//加锁操作
public void lock() {
     sync.lock();
}
```

默认构建非公平锁(关于公平锁与非公平锁的比较后面会再次提到)，同时可以通过参数来构建公平锁。

#### 加锁操作

```java
/**
 * 非公平锁实现
 */
static final class NonfairSync extends Sync {
    //加锁
    final void lock() {
        //执行CAS操作，本质就是CAS更新state：
        //判断state是否为0，如果为0则把0更新为1，并返回true否则返回false
        if (compareAndSetState(0, 1))
           //成功则将独占锁线程设置为当前线程  
          setExclusiveOwnerThread(Thread.currentThread());
        else
            //否则再次请求同步状态
            acquire(1);
    }
}
```

也就是说，通过CAS机制保证并发的情况下只有一个线程可以成功将state设置为1进而获取到锁，其他线程在执行cas的时候发现state不等于0进而执行acquire(1)。

#### 入同步队列

```java
public final void acquire(int arg) {
    //再次尝试获取同步状态
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

这里传入的arg是state的值，因为要获取锁，state为0的时候是释放锁，所以这里传入的值为1 ，进入方法后会先执行tryAcquire方法(1)方法，该方法是由ReetrantLock类的内部类实现的，具体如下

```java
//NonfairSync类
static final class NonfairSync extends Sync {

    protected final boolean tryAcquire(int acquires) {
         return nonfairTryAcquire(acquires);
     }
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    
    // ......
    
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        // 获取volatile类型的state
        int c = getState();
        if (c == 0) {
            /** 这里便是非公平锁的实现，state为0表示这个时刻没有其他线程获取锁
             *  此时不管队列的顺序，只要有线程获取锁失败则会执行cas操作再次尝试获取一次
             */
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 这里表示获取锁的线程是Owner，也就是重入锁重复加锁的情况，这时state会再加1
        // 因为是同一个线程，所以不会有线程安全的问题
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    // ......
}
```

仔细看源代码我们会知道，公平锁的tryAcquire是在其子类FairSync中实现的，那为什么非公平锁的实现需要提到父类中去实现呢？

因为ReetrantLock还实现了Lock接口，在Lock接口的语义里，tryLock()是不分公平与非公平的，（实际语义其实等同于非公平的）所以初始构造ReentrantLock时候，无论是new的fairSync还是notfairSync，都应该可以执行tryLock()，因此tryLock()用到的Sync(AQS)的nonfairTryAcquire()，应该在NonfairSync,FairSync父类中实现，就是此类Sync，这样才可以达到共用的效果。

假设有三个线程：线程1已经获得了锁，线程2正在同步队列中排队，此时线程3执行lock方法尝试获取锁时，线程1正好释放了锁，将state更新为0，那么线程3就可能在线程2还没有被唤醒之前获取到这个锁。

如果此时还没有获取到锁(nonfairTryAcquire返回false)，那么接下来会把该线程封装成Node对象去同步队列里排队，代码层面上是执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，传入Node.EXCLUSIVE表示该锁是独占锁。

```java
private Node addWaiter(Node mode) {
    //将请求同步状态失败的线程封装成结点
    Node node = new Node(Thread.currentThread(), mode);

    Node pred = tail;
    //如果是第一个结点加入肯定为空，跳过。
    //如果非第一个结点则直接执行CAS入队操作，尝试在尾部快速添加
    if (pred != null) {
        node.prev = pred;
        //使用CAS执行尾部结点替换，尝试在尾部快速添加
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //如果第一次加入或者CAS操作没有成功执行enq入队操作
    enq(node);
    return node;
}
```

如果第一次获取到锁，AQS还没有初始化，则tail为空，那么将执行enq(node)操作，如果非第一个节点，直接尝试使用cas操作加入队尾，如果cas操作失败或者说是第一次加入同步队列还是会执行enq(node)，继续看enq(node)。

