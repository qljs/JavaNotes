1. 实例对象存储在堆：对象内存在堆区，实例的引用在栈上，实例的元数据(class)在方法区/元空间。



对象不一定在堆区，有逃逸行为后，可能会在栈上。



2. 重量级锁：涉及到用户态到内核态的转变



3. 锁清除、

   

4. 锁粗化、

   

5. 逃逸分析、

   

6.  锁的膨胀升级





## 1. synchronized原理

​		`synchronized`内置锁是一种对象锁，是基于JVM内置锁来实现的，基于内部对象**Monitor**（监视器锁）的进入和退出实现同步功能，监视器锁依赖底层操作系统的Mutex lock（互斥锁），它是一个重量级的锁。

​		`synchronized`一般用来修饰方法或代码块。

```java
public class SynDemo {

    private final Object obj = new Object();

    public void methodA(){
        synchronized (obj){
            System.out.println("methodA");
        }
    }

    public synchronized void methodB(){
        System.out.println("mehtodB");
    }
}
```

​		反编译结果：

```java
public class com.source.code.SynDemo
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   // 省略.....
{
  public com.source.code.SynDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    // 省略

  public void methodA();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field obj:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        10: ldc           #5                  // String methodA
        12: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        15: aload_1
        16: monitorexit
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit
        23: aload_2
        24: athrow
        25: return
      // 省略

  public synchronized void methodB();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    // 省略
}
```

​		可以看到当`synchornized`修饰代码块时，通过`monitorenter`和`monitorexit`两个指令来完成加锁和退出，而当修饰方法时，通过标志`flags: ACC_SYNCHRONIZED`来实现方法的同步，当调用该方法时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取对象的monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。



### 1.2 对象内存布局

在JVM中对象布局分为三块：

- **对象头**：比如 hash码，对象所属的年代，对象锁，锁状态标志，偏向 锁（线程）ID，偏向时间，数组长度（数组对象）等
- **实例数据**：对象中成员变量，方法等
- **对齐填充**：JVM中规定对象的大小必须是8字节的整数倍

![](..\images\bf\obj.png)



> #### 对象头

Hotspot的对象头主要包括两部分数据：**Mark Word**（标记字段）、**Klass Pointer**（类型指针）。

- **Mark Word**：默认存储**对象的HashCode**，**分代年龄**和**锁标志位信息**。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。

- **Klass Point**：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。



> #### Mark Word

Mark Word在64位虚拟机中的存储内容：

![](D:\JavaNotes\JavaNotes\images\bf\Hotspot-dxt.png)



## 2. 锁升级

​		在JDK1.6之后，对JVM内置锁进行了重大优化，引入锁消除、锁粗化、轻量级锁、偏向锁、适应性自旋等等。锁的状态共有四种，分别为：**无锁状态**、**偏向锁**、**轻量级锁**和**重量级锁**。随着锁的竞争，锁会从偏向锁升级为轻量级锁，再升级为重量级锁，且锁的升级时单向的。

#### 2.1 偏向锁

​		在大多数情况下，锁总是由同一线程多次获得，不存在多线程竞争，，所以出现了偏向锁。其目标就是在只有一个线程执行同步代码块时能够提高性能。

​		当同一线程访问同步代码块并获取锁时，锁会转换为偏向锁状态，同时在Mark Word中存储偏向锁的线程ID，在线程再次请求获取锁时，不在做同步操作（CAS），而是与Mark Word中存储偏向锁的线程ID进行比较。

​		只有在其他线程尝试获取锁时，持有锁的线程才会释放锁，线程不会主动释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态。撤销偏向锁后恢复到无锁（标志位为“01”）或轻量级锁（标志位为“00”）的状态。



#### 2.2 轻量级锁

​		当锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。



#### 2.3 重量级锁

​		如果自旋超过一定次数，那么轻量级锁就会升级为重量级锁。升级为重量级锁后，所有的线程都将进入阻塞状态。



> #### 锁消除

- 锁消除：在下面的例子中，代码块中的`new Object()`不属于共享变量，JVM会进行优化，消除这种没有必要的锁。

```java
public void demo(){
    synchronized (new Object()) {
        // do something
    }
}
```



## 3. 逃逸分析

​		逃逸分析（Escape Analysis）简单来讲就是，Java Hotspot 虚拟机可以分析新创建对象的使用范围，**并决定是否在 Java 堆上分配内存的一项技术**。

**全局逃逸（GlobalEscape）**：即一个对象的作用范围逃出了当前方法或者当前线程，有以下几种场景：

- 对象是一个静态变量
- 对象是一个已经发生逃逸的对象
- 对象作为当前方法的返回值

**参数逃逸（ArgEscape）**：即一个对象被作为方法参数传递或者被参数引用，但在调用过程中不会发生全局逃逸，这个状态是通过被调方法的字节码确定的。



使用逃逸分析，编译器可以对代码做如下优化：

一、同步省略。如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步。

二、将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。

三、分离对象或标量替换。有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。







参考：

https://tech.meituan.com/2018/11/15/java-lock.html

https://zhuanlan.zhihu.com/p/69136675