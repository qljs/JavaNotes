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
        // 节点需要被唤醒
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

        // 等待队列中的后继节点，如果当前节点是共享的，那么这个字段是一个SHARED常量，
        // 也就是说节点类型(独占和共享)和等待队列中的后继节点共用同一个字段。
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

`ReentrantLock`默认使用的非公平锁，如果要是用公平锁，可以在创建对象时传入参数`true`，且`ReentrantLock`时独占模式。

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
    // h != t：队列中至少存在一个节点，node是final类型，可以直接通过==比较
    return h != t &&
        // 头节点next为空或者队列中第一个阻塞线程不等于当前线程
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```



#### 3.1.2 放入队列：addWaiter(Node.EXCLUSIVE)

**此时传入的独占类型的**

> #### addWaiter()

```java
private Node addWaiter(Node mode) {
    // 新建一个节点，此时节点的状态waitStatus是默认的0
    Node node = new Node(Thread.currentThread(), mode);
    // 获取尾结点
    Node pred = tail;
    if (pred != null) {
        // 尾结点不为空，将新建节点的前节点更新为原尾结点
        // 先更新新节点的前节点，即使下面CAS操作失败，也不会影响队列
        node.prev = pred;
        // 通过CAS操作尝试将新节点更新为尾结点
        if (compareAndSetTail(pred, node)) {
           	// 更新成功后，将原尾结点的next节点更新为新尾结点
            pred.next = node;
            return node;
        }
    }
    // 尾结点为null，代表整个队列还初始化，那个接下来就要初始化队列，更将节点插入队列
    enq(node);
    return node;
}
```



> #### 将节点插入队列：enq()

```java
private Node enq(final Node node) {
    // 死循环，要么成功，要么一直尝试
    for (;;) {
        // 获取尾结点
        Node t = tail;
        // 再次判断尾结点是否为空
        if (t == null) {
            // 尾结点为空，代表没有被他线程抢先初始化队列
            // 通过CAS操作设置头结点，注意头结点是一个空的node
            if (compareAndSetHead(new Node()))
                // 头尾节点指向同一个node
                tail = head;
            	// 未终止循环，下次循环会走else节点
        } else {
            // 尾结点不为空
            node.prev = t;
            // 通过CAS操作将新节点更新为尾结点
            if (compareAndSetTail(t, node)) {
                // 更新成功，将原尾结点的next节点更新为新尾结点
                t.next = node;
                // 返回最近进入队列的节点
                return t;
            }
        }
    }
}
```



#### 3.1.3 阻塞队列中的线程节点：acquireQueued()

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取新建节点的前节点
            final Node p = node.predecessor();
            // 新建节点的前节点是头结点，尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 将新建节点更新为头结点，就是把thread和prev置空
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 新建节点的前节点不是头结点，查看前节点是否要被唤醒、移除、或者设置为等待被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 阻塞线程
                parkAndCheckInterrupt())
                interrupted = true;
            	// 这里没有终止循环，在线程被唤醒后会不断循环尝试获取锁
        }
    } finally {
        // 在tryAcquire()抛出异常时，failed会为true
        if (failed)
            cancelAcquire(node);
    }
}
```



> 检查并更新节点状态：shouldParkAfterFailedAcquire()

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 前节点状态为-1，需要被唤醒
        return true;
    if (ws > 0) {
        // 从队列中移除需要被取消的节点，直至找到一个可以被唤醒的节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 将前节点状态设置为等待被唤醒，进入这个分支的只剩下0，-2，-3了，状态0是节点初始化时候的状态
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```





> #### 阻塞线程：parkAndCheckInterrupt()

```java
/**
* 阻塞当前节点，返回当前Thread的中断状态
* LockSupport.park 底层实现逻辑调用系统内核功能 pthread_mutex_lock 阻塞线程
*/
private final boolean parkAndCheckInterrupt() {
    // 阻塞
    LockSupport.park(this);
    // 线程是否中断
    return Thread.interrupted();
}
```



### 3.1.4 加锁流程图

![](../images/bf/fairsyn.jpg)





### 3.2 解锁

```java
public void unlock() {
    // 释放锁
    sync.release(1);
} 

public final boolean release(int arg) {
    // 是否可以释放锁
    if (tryRelease(arg)) {
        Node h = head;
        // 在上面节点拆入队列时，做过一次前驱节点状态更新为-1的操作，头结点的状态就是在那里更新的
        if (h != null && h.waitStatus != 0)
            // 唤醒线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



#### 3.2.1 尝试释放锁：tryRelease()

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 当前线程不是持有锁的线程，抛错
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否可以释放锁
    boolean free = false;
    if (c == 0) {
        // C==0,线程重入，等于0时可以释放
        free = true;
        // 独占线程清空
        setExclusiveOwnerThread(null);
    }
    // 不等于0，更新线程重入次数
    setState(c);
    return free;
}
```



> #### 唤醒线程：unparkSuccessor()

```java
private void unparkSuccessor(Node node) {
    // node是头节点
    int ws = node.waitStatus;
    if (ws < 0)
        // 将头节点的状态更新为0
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    // 头节点的后继节点为null，或者需要被取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 头节点为null时，尾节点也一定为空，不会进入该循环
        // 从后向前循环找出第一个状态waitStatus小于等于0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒线程
        LockSupport.unpark(s.thread);
}
```



## 4.  Semaphore源码分析

`Semaphore`：信号量，用于控制访问资源的线程数量，可以用来做单机限流等。

```java
// permits：同时可获去资源的线程数，默认采用非公平锁
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

// fair：true:公平锁；false：非公平锁
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}

```

使用例子：

```java
public class SemaphoreDemo {

    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);

        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "拿到资源！");
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();
                }
            },"线程" + i).start();
        }
    }
}
```



### 4.1 阻塞并获取资源：acquire()

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 判断线程是否被中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // 获取共享资源
    if (tryAcquireShared(arg) < 0)
        // 允许获取数量小于零，将新的线程放入队列阻塞
        doAcquireSharedInterruptibly(arg);
}
```



> #### 尝试获取共享资源：tryAcquireShared()

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    // 系循环，不断尝试，直到达到允许线程数
    for (;;) {
        // state：共享资源允许线程访问的数量
        int available = getState();
        int remaining = available - acquires;
        // 剩余数量小于0
        if (remaining < 0 ||
            // CAS更新数量
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```



### 4.2 未拿到资源的线程阻塞：doAcquireSharedInterruptibly()

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 插入队列，与ReectrantLock逻辑一样，但是模式变为了共享
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                // 前驱节点时头节点，即自己时第一个阻塞的节点，那么就去尝试获取资源
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 拿到资源
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 与ReectrantLock逻辑一样
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 前驱节点状态为-1，代表已经有其他节点更新过了前驱节点的状态，同时当前线程已经中断
   				// 那么队列中该节点的位置被别人抢了
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



## 5. CountDownLatch()

​		CountDownLatch允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

​		CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当 一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的 线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

​		CountDownLatch的场景：例如LOL或者王者荣耀游戏开始是，需要玩家全部准备好，才能进入游戏。

```java
public class CountDownLatchDemo {

    public static void main(String[] args) {
        Random random = new Random();
        CountDownLatch latch = new CountDownLatch(5);
        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + "载入中！");
                try {
                    Thread.sleep(random.nextInt(5000));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "加载完毕！");
                latch.countDown();
            },"玩家" + i).start();
        }

        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("开始游戏！！");
    }
}
```





## 6. CyclicBarrier

​		栅栏屏障，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程 到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

​		CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。

​		**CountDownLatch的实现是基于AQS的，而CycliBarrier是基于 ReentrantLock(ReentrantLock也属于AQS同步器)和 Condition 的。**

```java
/**
* CyclicBarrier中两个变量count和parties
* count：还在等待的线程数
* parties：传入的总线程数
*/
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        Random random = new Random();
        CyclicBarrier cyclicBarrier = new CyclicBarrier(6);
        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + "进入游戏！");
                try {
                    Thread.sleep(random.nextInt(6000));
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            },"玩家" + i).start();
        }
        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println("开始游戏！");
    }
}
```





## 参考

https://www.javadoop.com/post/AbstractQueuedSynchronizer-2