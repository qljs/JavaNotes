[TOC]

## 1. Unsafe

​		Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。

​		Unsafe类为单例实现，提供静态方法getUnsafe获取Unsafe实例，当且仅当调用getUnsafe方法的类为引导类加载器所加载时才合法，否则抛出SecurityException异常。

> #### 获取Unsafe

1. 把调用Unsafe相关方法的类Demo所在jar包路径追加到默认的bootstrap路径中，使得A被引导类加载器加载`java -Xbootclasspath /Demo:${path}`；
2. 通过反射获取

```java
public class UnsafeUtil {

    public static Unsafe getUnsafe(){
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            // theUnsafe：private static final Unsafe theUnsafe;
            return (Unsafe) field.get(null);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```





## 2. Atomic

​		Atomic 是指一个操作是不可中断的操作，Atomic包里的类基本都是使用`Unsafe`实现的包装类。根据操作的数据类型，可以将JUC包中的原子类分为4类：

### 2.1基本类型

- AtomicInteger：整型原子类。
- AtomicLong：长整型原子类。
- AtomicBoolean ：布尔型原子类。

三种类型方法基本相似，以`AtomicInteger`中主要方法为例：

```java
public final int get(); // 获取当前值
public final int getAndSet(int newValue); // 设置新值并返回旧值
public final boolean compareAndSet(int expect, int update); // 输入expect值等于预期值，则更新为新值update
public final int getAndIncrement(); // 获取当前值并自增
public final int getAndDecrement(); // 获取当前值并自减
public final void lazySet(int newValue); // 最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
public final int getAndAdd(int delta); // 先获取当前值，然后加上delta
```



### 2.2 数组类型

- AtomicIntegerArray：整型数组原子类。
- AtomicLongArray：长整型数组原子类。
- AtomicReferenceArray ：引用类型数组原子类。

`AtomicIntegerArray`中主要方法：

```java
public final int get(int i) //获取 index=i 位置元素的值
public final int getAndSet(int i, int newValue)//返回 index=i 位置的当前的值，并将其设置为新值：newValue
public final int getAndIncrement(int i)//获取 index=i 位置元素的值，并让该位置的元素自增
public final int getAndDecrement(int i) //获取 index=i 位置元素的值，并让该位置的元素自减
public final int getAndAdd(int i, int delta) //获取 index=i 位置元素的值，并加上预期的值
boolean compareAndSet(int i, int expect, int update) //如果输入的数值等于预期值，则以原子方式将 index=i 位置的元素值设置为输入值（update）
public final void lazySet(int i, int newValue)//最终 将index=i 位置的元素设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

**将数组传入AtomicIntegerArray中时，会把传入的数组clone一份，当AtomicIntegerArray修改数组中的值时，不会影响原数组**

```java
public AtomicIntegerArray(int[] array) {
    // Visibility guaranteed by final field guarantees
    this.array = array.clone();
}
```



### 2.3 引用类型

- AtomicReference：引用类型原子类。
- AtomicMarkableReference：原子更新带有标记的引用类型。
- AtomicStampedReference ：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

**CAS**：CAS (compare and swap)，原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。

> ABA问题

​		第一个线程拿到了变量值A，之后执行业务逻辑，此时，第二个线程也拿到了变量值A，然后先将变量值该为B，之后又改为A，此时第一个线程执行完业务逻辑，去修改变量的值时，并不会发现变量A被修改过。

​		解决ABA常用的办法就是加上版本号，每次修改变量值的时候，用旧值和版本号一起比较，成功后修改变量值并更新版本号，例如每次成功后版本号加一。



> **AtomicReference常用方法及例子**

```java
public class AtomicReferenceDemo {

    public static void main(String[] args) {
        AtomicReference<User> reference = new AtomicReference<User>();
        User user1 = new User("张三",10);
        User user2 = new User("李四",20);
        // 设置一个新值
        reference.set(user1);
        // 若传入值等于预期值，则更新为新值
        reference.compareAndSet(user1,user2);
        // 获取当前值
        User user = reference.get();
        System.out.println(user.toString());

    }
}
```



> **AtomicStampedReference常用方法和例子**

```java
public class AtomicStampedReferenceDemo {

    public static void main(String[] args) {
        User user1 = new User("张三", 10);
        // 第一个参数：初始值；第二个参数：初始版本号
        AtomicStampedReference reference = new AtomicStampedReference(user1,0);
        int oldStamp = reference.getStamp();

        Thread thread = new Thread(() -> {
            User otherUser = new User("李四", 20);
            reference.compareAndSet(user1, otherUser, 0, 1);
            System.out.println("子线程修改：" + reference.getReference());
            reference.compareAndSet(otherUser, user1, 1, 2);
            System.out.println("子线程还原：" + reference.getReference());
        });
        thread.start();
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        User user2 = new User("李四", 20);
        boolean result = reference.compareAndSet(user1, user2, oldStamp, oldStamp + 1);
        if(result){
            System.out.println("修改成功");
        } else {
            System.out.println("修改失败");
        }
    }
}
```



### 2.4 对象属性修改

- AtomicIntegerFieldUpdater:原子更新整型字段的更新器。
- AtomicLongFieldUpdater：原子更新长整型字段的更新器。
- AtomicReferenceFieldUpdater：原子更新引用类型里的字段。

> **AtomicIntegerFieldUpdater例子**

```java
public class AtomicIntegerFieldUpdaterDemo {

    public static void main(String[] args) {
        // age不能用private修饰，且必须用volatile修饰
        AtomicIntegerFieldUpdater<User> updater = AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
        User user = new User("张三", 18);
        updater.compareAndSet(user,18,30);
        System.out.println(updater.get(user));
        updater.incrementAndGet(user);
        System.out.println(updater.get(user));

    }
}
```

