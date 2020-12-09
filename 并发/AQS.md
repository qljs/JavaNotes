[TOC]

# AQS及AQS组件





## 1. AQS简介

​		AQS全程为AbstractQueuedSynchronizer，位于java.concurrent.util包，它定义了一套多线程访问共享资源的同步器框架，是一个依赖状态（state）的同步器，JUC中大多数同步器实现都是围绕等待队列、条件队列、独占、共享等行为，而AQS就是对这种行为的抽象。		



> #### 队列

AQS中定义了两种队列，分别是：

- **同步队列**：基于双向链表，FIFO的数据结构队列，Java中的CLH队列是原CLH队列的一个变种，线程有原来的自旋变为了阻塞。
- **条件队列**：是一个多线程间协调通信的工具类，使得某个或者某些线程一起等待某个条件（Condition），只有当该条件具备时 ，这些等待线程才会被唤醒，从而重新争夺锁。



> #### 公平锁和非公平锁

- **公平锁**：新线程会进入等待队列中，等待被唤醒，遵循FIFO。

  优点是所有线程都能获得资源，不会被饿死；缺点是吞吐量会下降，队列里除了第一个线程其他都会被阻塞，CPU唤醒线程消耗较大。

- **非公平锁**新线程会先尝试获取锁，若失败进入等待队列，与公平锁相同，CPU需要唤醒的线程+1；若成功，则直接获取锁。

  优点是可以减少CPU唤醒线程消耗，提高吞吐量；缺点是队列中的线程可能会被饿死。



> #### 重入锁和不可重入锁

- **可重入锁**：同一个线程可以多次获取之前已经获取到的锁。
- **不可重入锁**：同一个线程只能获取一次锁。



> #### 读锁和写锁

- **读锁（共享锁）**：该锁可以被多个线程获取。如果一个线程对数据加上共享锁，其他线程只能对数据加共享锁。
- **写锁（排他锁）**：指该锁一次只能被一个线程持有。被加上写锁的数据不能被其他线程加任何类型的锁。



## 2. AbstractQueuedSynchronizer

`AbstractQueuedSynchronizer`中重要的属性

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    // 定义了一个内部类Node
    static final class Node {
        // 标记节点为共享模式
        static final Node SHARED = new Node();
        // 标记节点为独占模式
        static final Node EXCLUSIVE = null;

        // 在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待
        static final int CANCELLED =  1;
        // 后继节点需要被唤醒
        static final int SIGNAL    = -1;
        // 节点在等待队列中，节点的线程等待在Condition上，当其他线程对Condition调用了signal()方法后，
    	// 该节点会从等待队列中转移到同步队列中，加入到同步状态的获取中
        static final int CONDITION = -2;
        // 表示下一次共享式同步状态获取将会被无条件地传播下去
        static final int PROPAGATE = -3;

        // 存储上面几个状态，使用volatile关键字，保证线程间的可见性
        volatile int waitStatus;
		
        // 前节点
        volatile Node prev;
        // 后节点
        volatile Node next;

        // 节点中的线程
        volatile Thread thread;

        // 链接到等待条件的下一个节点，或者链接到特殊值SHARED。
        Node nextWaiter;

       // .......
    }

    // 头节点，当队列不为空时（为了处理一些判断），头节点是一个空的节点，next指向的才时队列中第一个被阻塞的线程
    private transient volatile Node head;

    // 尾节点
    private transient volatile Node tail;

    // 记录锁被重入的次数
    private volatile int state;
   
}   
```





## 3. ReentrantLock公平锁源码

`ReentrantLock`默认使用的非公平锁，如果要是用公平锁，可以在创建对象时传入参数`true`。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

公平锁`FairSync`和非公平锁`NonfairSync`都是通过中`Sync`来实现的，而`Sync`继承了`AbstractQueuedSynchronizer`。



### 3.1 加锁

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }
    
    /**
    * 调用父类的方法， 
    */
    public final void acquire(int arg) {
        // tryAcquire()：尝试获取锁，成功返回true
    	if (!tryAcquire(arg) &&
            // acquireQueued()：挂起未获取到锁的线程并放入队列，成功返回true
        	acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 阻塞当前节点，返回当前Thread的中断状态
        	selfInterrupt();
	}
}
```



#### 3.1.1 尝试获取锁：tryAcquire()

```java
protected final boolean tryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取锁重入次数
    int c = getState();
    if (c == 0) {
        // 当前没有线程持有锁
        // hasQueuedPredecessors()：判断当前队列是否为空，因为是公平锁队，列不为空时，当前线程会进入队列
        if (!hasQueuedPredecessors() &&
            // 通过CAS操作尝试将state更新为acquires，若成功代则线程拿到锁
            compareAndSetState(0, acquires)) {
            // exclusiveOwnerThread：继承自AbstractOwnableSynchronizer，用于存放持有锁的线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 线程重入，更新次数state
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



> #### 判断阻塞队列是否为空：hasQueuedPredecessors()

```java
public final boolean hasQueuedPredecessors() {
   
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    // h != t：队列中至少存在一个节点，队列中有节点时，头节点是一个空的节点，只有next有值
    return h != t &&
        // 头节点next为空或者队列中第一个阻塞线程不等于当前线程
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```



#### 3.2.2 线程挂起，放入队列：acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

> #### addWaiter()

```java

```





## 参考

https://www.javadoop.com/post/AbstractQueuedSynchronizer-2