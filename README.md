





`List`应该是工作中常使用的集合之一，其中使用较多的是`ArrayList`和`LinkedList`，这两个 List 的区别面试中也常会被问到，还是有必要熟练掌握的。



## 一、ArrayList的核心源码解析

### 1. 构造方法

`ArrayList`底层使用的是数组，查找数据快、增删数据慢。

先来看下`ArrayList`的主要属性和构造方法。可以看到，在ArrayList中，定义了两个空数组：`EMPTY_ELEMENTDATA`和`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`。

为什么要定义两个空数组呢？试想下，如果只定义一个，那么构造方法传入的是0，在初始化的时候，是采用默认的长度，还是传入的0，而且扩容的时候，又要怎么选择呢？

```java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    // 默认的初始化长度
    private static final int DEFAULT_CAPACITY = 10;
    // 长度设置为0时，初始化的数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    // 无参构造默认初始化的数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 存储数据的数组
    transient Object[] elementData;
    // 对数组增删的次数
    protected transient int modCount = 0;

    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        }
    }

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
    
    public ArrayList(Collection<? extends E> c) {
        Object[] a = c.toArray();
        if ((size = a.length) != 0) {
            if (c.getClass() == ArrayList.class) {
                elementData = a;
            } else {
                elementData = Arrays.copyOf(a, size, Object[].class);
            }
        } else {
            // replace with empty array.
            elementData = EMPTY_ELEMENTDATA;
        }
    }
}
```



### 2. 添加/删除方法

> #### 添加方法

添加方法有两种，一种是在末尾添加，一种是插入到指定位置

```java
public boolean add(E e);
public void add(int index, E element);
```



**在末尾添加元素**

```java
/**
  * 向数组末尾添加元素.
  *
  * @param e element to be appended to this list
  * @return <tt>true</tt> (as specified by {@link Collection#add})
  */
public boolean add(E e) {
    // 这个方法中做了两件事；
    // 一是在插入元素时，判断初始化时，用的是否是无参构造，二是判断新元素加入后，是否需要扩容
    ensureCapacityInternal(size + 1);
    // size是元素数量；新元素添加到末尾
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

/**
 * 获取数组的容量
 */ 
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 若实例化采用的无参构造，则从minCapacity默认容量中选择一个大的
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

// 判断是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```



