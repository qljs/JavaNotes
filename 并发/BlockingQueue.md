[TOC]



## 1. BlockingQueue简介

​		`BlockingQueue`阻塞队列，通常用链表或数组实现，主要有`FIFO`和`LIFO`两种，其实现是基于**AQS（CLH+条件队列+ReentrantLock）**。最经典的应用场景就是生产者-消费者。

其主要方法及参数如下：

```java
// ********* 入队 ***********
boolean add(E e); // 向队列尾插入元素，若队列已满，则抛错
boolean offer(E e); // 向队列尾插入元素，若队列已满，返回fasle
void put(E e) throws InterruptedException; // 向队列尾插入元素，若队列已满，则阻塞
//  ********* 出队 ***********
E take() throws InterruptedException; // 从队列头取出元素，若没有则阻塞
E poll(long timeout, TimeUnit unit) throws InterruptedException; // 从队列头取出元素，等待指定时间
int drainTo(Collection<? super E> c);  // 将队列中的全部元素放入指定集合中
```

`BlockingQueue`常用的实现类：

- `ArrayBlockingQueue`：基于数组的有界队列；
- `LinkedBlockingQueue`：基于链表的可选有界队列；
- `PriorityBlockingQueue`：基于堆的无界优先级队列，优先级通过传入`Comparator`实现；
- `DelayQueue`：基于时间的调度，优先级堆支持的队列，内部基于无界队列PriorityQueue实现；



## 2. BlockingQueue源码

以`ArrayBlockingQueue`为例，分析BlockingQueue实现原理。

> #### 构造方法及重要属性

```java
// capacity：items的容量；fair：是否使用公平锁
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    // items：存放入队放入插入的元素
    this.items = new Object[capacity];
    // 新建锁，默认非公平
    lock = new ReentrantLock(fair);
    // 处理出队的队列
    notEmpty = lock.newCondition();
    // 处理入队的队列
    notFull =  lock.newCondition();
}

// 内部类
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        // 条件队列的首节点
        private transient Node firstWaiter;
        // 条件队列的尾节点
        private transient Node lastWaiter;
}    
```



### 2.1 入队：put()

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 尝试获取锁，未获取到锁的放入同步队列
    lock.lockInterruptibly();
    try {
        // count：items中已有的元素数量，items.length：初始化时传入的数量
        while (count == items.length)
            // items中已满，notFull队列被阻塞
            notFull.await();
        // 入队
        enqueue(e);
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```



#### 2.1.1 尝试获取锁，或放入同步队列：lockInterruptibly()

```java
// 尝试获取锁
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // tryAcquire()：ReentrantLock中的方法
    if (!tryAcquire(arg))
        // 未获取到锁，放入同步队列，阻塞线程
        doAcquireInterruptibly(arg);
}
```

​		`tryAcquire()`在AQS中`ReentrantLock`源码分析过，而`doAcquireInterruptibly()`也和其中的`acquireQueued()`方法相差无几，[AQS及其组件](AQS.md)。

​		入队方法第一步会通过`lock.lockInterruptibly()`方法先去争抢锁，对于没有获取到锁的线程，会被放入等待队列中，接下来，会先判断数组`items`是否已满，若满了则进入`await()`方法。



#### 2.1.2  放入条件队列：await()

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 放入条件队列
    Node node = addConditionWaiter();
    // 释放锁，并返回持有锁时重入次数，此时其他已经已经可以重新争抢锁了
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 等待被放入同步队列
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // 判断线程是否中断，不等于0时，代表中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 走到这里证明已经放入同步队列中或中断了。
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        // 线程被阻塞且不是因为异常中断，将interruptMode状态置为REINTERRUPT（1）
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        // 后继节点不为空，清理条件队列中需要移除的节点
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        // 重新中断线程或抛出异常
        reportInterruptAfterWait(interruptMode);
}
```

**这里中断线程调用的都是Thread.interrupted()，而interrupted()并不会让线程立刻停止运行。**



> #### 放入条件队列：addConditionWaiter()

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    if (t != null && t.waitStatus != Node.CONDITION) {
        // 当尾节点状态不为Node.CONDITION时，检查一次条件队列，剔除状态不为CONDITION的节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 新建一个节点，插入条件队列最后
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    // 插入完成，更新尾节点
    lastWaiter = node;
    return node;
}

/**
* 删除取消的节点
*/
private void unlinkCancelledWaiters() {
    // t：在下面的循环中，记录下一次要检查的节点
    Node t = firstWaiter;
    // trail：在循环中记录到当前检查节点之前，最后一个正常的节点
    Node trail = null;
    // 头节点不为空，即条件队列不为空
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            // 当前节点状态不等于CONDITION，将其从队列中移除
            // 先将其后继节点置为null
            t.nextWaiter = null;
            // trail为null时，即还没找到正常的节点
            if (trail == null)
                // 头节点向后移动
                firstWaiter = next;
            else
                // 进入这一步，在之前找到了正常节点，并由trail记录
                // 先将上一个正常节点的后继节点更新为当前节点的后继节点，即将当前节点从队列中移除
                trail.nextWaiter = next;
            if (next == null)
                // 下一个节点为空，队列检查完毕，将最后一个正常节点赋值给尾节点
                lastWaiter = trail;
        }
        // 当前节点状态正常（等于Node.CONDITION）
        else
            trail = t;
        t = next;
    }
}
```



> #### 释放当前线程持有的锁：fullyRelease()
>
> 在将线程放入条件队列之后，就会释放线程持有的锁，进入到`fullyRelease()`方法。

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取线程重入次数
        int savedState = getState();
        // 释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            // 释放锁失败，将状态改为CANCELLED（1）
            node.waitStatus = Node.CANCELLED;
    }
}

/**
* 释放锁
*/
public final boolean release(int arg) {
    // 释放锁
    if (tryRelease(arg)) {
        // arg：时是上面getState()获取到的，所以正常情况下会走进里面的分支
        Node h = head;
        // 头结点不为空，
        if (h != null && h.waitStatus != 0)
            // 给线程一个许可
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    // 剔除需要删除的节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 给后继节点上线程一个许可，之后调用park()方法时不会被阻塞
        LockSupport.unpark(s.thread);
}
```



> #### 是否被放入同步队列中：isOnSyncQueue()
>
> 线程首先进入条件队列，之后会在循环中不断判断是否进入了同步队列

```java
public final void await() throws InterruptedException {
    // 省略
    // 是否被放入同步队列
    while (!isOnSyncQueue(node)) {
        // 第一次循环时还未放入同步队列，会被阻塞
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 不等于0时，线程被中断了，也就没有必要一直循环判断了
            break;
    }
    // 省略
}

/**
* 是否被放入同步队列
*/
final boolean isOnSyncQueue(Node node) {
    // 第一次进来时，节点肯定还未放入同步队列，所以会返回false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // node的后继节点不等于空时，证明放入了肯定已经放入了同步队列
    if (node.next != null) 
        return true;
    // 检查是否在同步队列中，因为有可能被中断从队列中移除了
    // 在同步队列自后向前查找
    return findNodeFromTail(node);
}
```



> #### 检查在wait()方法中是否被中断：checkInterruptWhileWaiting(node)

```java
/**
* interruptMode：各个状态的含义
* 0：什么都不做，没有被中断过；
* THROW_IE：await()方法抛出InterruptedException异常，因为它代表在await()期间发生了中断；
* REINTERRUPT：重新中断当前线程，因为它代表await()期间没有被中断，而是signal()以后发生的中断
*/
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
    0;
}

/**
* 进入此方法是线程已经被中断
*/
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        // 真正将节点放入队列中的方法在这里，
        // 虽然放入同步队列，但是此时还未从条件队列中移除
        enq(node);
        return true;
    }
	
    // CAS更新失败。
    while (!isOnSyncQueue(node))
        // 让出CPU资源，进入就绪状态
        Thread.yield();
    return false;
}
```



#### 2.1.3 移入等待队列：enqueue()

在数组`items`中还未满时，会走到这个分支，需要注意的是。

```java
private void enqueue(E x) {
    final Object[] items = this.items;
    // 将元素放入items
    items[putIndex] = x;
    if (++putIndex == items.length)
        // 索引与items长度相等，即items中已满，putIndex归零
        putIndex = 0;
    // items中的数量+1
    count++;
    // 移入同步队列
    notEmpty.signal();
}
```



> #### 移入同步队列：signal()

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获取条件队列的头节点
    Node first = firstWaiter;
    if (first != null)
        // 真正的转移方法
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        // 第一次进来时节点first就是头节点firstWaiter，即从头节点不断向后找
        if ( (firstWaiter = first.nextWaiter) == null)
          	// firstWaiter为空，代表着已经找完了，那么就将尾节点更新为空
            lastWaiter = null;
        // 最后一个节点的next指向空
        first.nextWaiter = null;
        // 把节点从条件队列转移到同步队列中
    } while (!transferForSignal(first) &&
             // 后继节点不为空，为空也没必要往后继续找了
             (first = firstWaiter) != null);
}


/**
* 先看一下判断循环是否结束的第一个条件transferForSignal()
*/
final boolean transferForSignal(Node node) {
 	// 先尝试将节点状态更新为0，接下来要放入同步队列
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        // 节点状态更新失败，即其他线程已经更成功或者出现其他错误，返回
        return false;

    // 将节点线程放入同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    // 节点线程状态大于零（取消状态），或者状态更新为-1（待唤醒）时失败
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```



#### 2.1.4 put()方法流程图

![](D:\JavaNotes\JavaNotes\images\bf\put.png)



### 2.2 出队：take()

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 与入队的逻辑一样
    lock.lockInterruptibly();
    try {
        while (count == 0)
            // 与入队调用的是同一个方法
            notEmpty.await();
        // 返回队列中的元素
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        // 清空迭代器
        itrs.elementDequeued();
    // 入队将条件队列节点移入同步队列方法，在这里会调用unpark()方法，唤醒条件队列中被阻塞的线程
    notFull.signal();
    return x;
}
```



