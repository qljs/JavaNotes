



## HashMap



### 1. Jdk1.7的HashMap

在Jdk1.7及之前，HashMap采用的是数组+链表的结构，其中比较重要的属性如下：

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable{
	// 数组的初始化大小，
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    // 数组最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // map中数据数量
    transient int size;
    // 扩容阈值：数组长度*负载因子
    int threshold;
   
    static final Entry<?,?>[] EMPTY_TABLE = {};
    // 存储数据的数组+链表
	transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
    // 对map操作的次数，在循环map时可以用来判断是否有删除元素
    transient int modCount;
    
    // Entry：用来存储的数据结构
	static class Entry<K,V> implements Map.Entry<K,V> {
        // map中的key
        final K key;
        // map中的value
        V value;
        // 下一个节点，正是利用这个next形成了链表
        Entry<K,V> next;
        // k的hash值
        int hash;

        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }
    }   
}    
```

​	

> #### put()方法源码分析

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        // 桶为空，初始化
        inflateTable(threshold);
    }
    if (key == null)
        // 若原来有key为null的，则替换，没有直接插入
        return putForNullKey(value);
    int hash = hash(key);
    // 计算出索引位置
    int i = indexFor(hash, table.length);
    // 桶当前位置有值，若当前链表存在key相等的，则进行替换
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    // 操作次数加+1
    modCount++;
    // 为发生hash碰撞，插入元素
    addEntry(hash, key, value, i);
    return null;
}

/**
* 添加元素
*/
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        // map中元素数据量大于阈值，当前索引位置上不为空，进行扩容
        resize(2 * table.length);
        // 计算出新插进来的元素索引位置
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    // 将元素插入map中
    createEntry(hash, key, value, bucketIndex);
}

/**
* 进行扩容
*/
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 原来桶的长度已经到最大值了，
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    // 新建一个数组
    Entry[] newTable = new Entry[newCapacity];
    // 转移数据
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    // 重新计算扩容阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

/**
* 转移数组
*/
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        // 循环链表，倒叙插入
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}

/**
* 插入元素，头插法，即新插入的元素放在数组上，原来的值推入链表中
*/
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```



> #### get()方法源码分析

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    // 获取Entry
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }
	// 计算hash值
    int hash = (key == null) ? 0 : hash(key);
    // 获取table索引位置上的值，根据key判断是否时同一个，若不是在链表上继续找
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```



### 2.  Jdk1.8的HashMap

​		在jdk1.8中，对hashMap进行了改进，引入了红黑树，结构变为了数组+链表+红黑树。当链表长度大于8（默认值）时，会将链表转换为红黑树，若数组长度小于64时，会优先进行扩容，而不是转化为红黑树，jdk1.8中HashMap的大多数属性与jdk1.7中一样。

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    /**
    * 省略与1.7中一样的书信
    */

    // 链表转红黑树的默认节点数
    static final int TREEIFY_THRESHOLD = 8;
    // 红黑树退回链表的默认节点数
    static final int UNTREEIFY_THRESHOLD = 6;
    // 链表转红黑树时数组最小长度
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存放数据的数组
    transient Node<K,V>[] table;

    // 链表节点结构
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    // 红黑树节点结构
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        // 父节点
        TreeNode<K,V> parent;  // red-black tree links
        // 左子节点
        TreeNode<K,V> left;
        // 右子节点
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        // 节点颜色
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
    }
}   
```

​	

> #### put()方法源码分析

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```







### 3. Jdk1.7中的死循环问题

模拟下1.7中死循环：

```java
public class HashMapDemo {

    private static HashMap<Integer, Integer> map = new HashMap<>();
    private static AtomicInteger integer = new AtomicInteger();

    public static void main(String[] args) {
        HashMapTest thread = new HashMapTest();
        HashMapTest thread2 = new HashMapTest();
        HashMapTest thread3 = new HashMapTest();
        HashMapTest thread4 = new HashMapTest();
        thread.start();
        thread2.start();
        thread3.start();
        thread4.start();
    }

    static class HashMapTest extends Thread{

        @Override
        public void run() {
            while (integer.get() < 1000000) {
                map.put(integer.get(),integer.get());
                integer.incrementAndGet();
            }
        }
    }
}
```



通过`jstack pid`查看堆栈信息，可以发现死循环发生在`transfer()`方法中，该方法作用是在扩容后，将数据转入新集合中。

![](..\images\connection\hashmap-jstack.png)

​	

接下先看下hashMap中`put()`的过程：

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

​		假设在并发情况下，有两个线程同时扩容，都走到了`transfer()`方法，线程1在执行完` Entry<K,V> next = e.next;`这一行，失去CPU时间片，此时**e指向e0，next指向e1**。

![](..\images\connection\start.png)



​		而线程2继续执行，线程2扩容完之后，根据缓存一致性协议，新的table最终结果如下图（假设扩容后索引位置未变）：

![](..\images\connection\t2-end.png)

​		此时线程1重新获取获取CPU执行权限，继续扩容，第一次循环结束，结果如图：

![](..\images\connection\t1-1.png)



​		next不为空，继续进行第二次循环：

![](D:\JavaNotes\JavaNotes\images\connection\end.png)









### 