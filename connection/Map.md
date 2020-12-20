



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

#### 1.1 put()方法源码分析

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
    // 发生hash碰撞，若当前链表存在key相等的，则进行替换
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



### 2.  Jdk1.8的HashMap

在jdk1.8中，对hashMap进行了改进，引入了红黑树，结构变为了数组+链表+红黑树。当链表长度大于8（默认值）时，会将链表转换为红黑树，若数组长度小于64时，会优先进行扩容，而不是转化为红黑树，jdk1.8中HashMap的大多数属性与jdk1.7中一样。

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    /**
    * 省略与1.7中一样的属性
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

#### 2.1 put()方法源码分析

```java
public V put(K key, V value) {
    // JDK1.8中计算hash值的方式变成了与低16位做^位运算
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // table未初始化时，进行初始化
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        // table当前索引位置为空，直接插入
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 与table索引位置值相同，替换
            e = p;
        else if (p instanceof TreeNode)
            // 发生hash碰撞，当前table节点之后为红黑树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 当前节点之后为链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 当前节点的后继节点为空，直接插入
                    p.next = newNode(hash, key, value, null);
                    // 链表长度超过阈值，若table长度未超过默认长度扩容，否则转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 替换节点
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
             // LinkedHashMap重写了该方法
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        // 扩容
        resize();
     // LinkedHashMap重写了该方法
    afterNodeInsertion(evict);
    return null;
}
```



> #### 扩容：resize()

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 旧表长度大于零，即已经初始化过
    if (oldCap > 0) {
        // 长度大于等于最大容量时，扩容临界值取Integer最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容两倍，小于最大容量
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 // 旧table长度大于16
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 扩容阈值也增加两倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0)
        // 旧table长度为0，新的长度设置为原扩容阈值
        newCap = oldThr;
    else {
        // oldCap和oldThr都为0，使用默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 新的扩容阈值为0，计算扩容阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 构建新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 旧map有数据，转移数据
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) { 
                oldTab[j] = null;
                if (e.next == null)
                    // 当前索引位置只有table节点上有值，即后面没有链表或者红黑树，计算在新table中的位置并赋值到新位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 节点是红黑树节点，调用红黑树的转移方法
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                     // 循环将旧table链表上数据转移到新的table中
                    // loHead：索引位置为0的链表头结点，
                    // loTail：索引位置为0的链表尾结点，
                    Node<K,V> loHead = null, loTail = null;
                    // hiHead：索引位置非0的链表头结点，
                    // hiTail：索引位置非0的链表尾结点，
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        // 可以看到1.8中，整个链表上的元素不会重新计算位置，位置有头结点决定
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                             // loTail：第一次进来，有值之后后面的节点都不断加入loTail后面
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 非零的位置的链表，新的位置为原位置+旧oldCap长度
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 旧table为空，直接返回新的数组
    return newTab;
}
```



> #### 扩容后红黑树节点重新计算在新数组中的位置：split()

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    // lc：记录与旧table中索引相同的节点个数，hc：记录旧table+oldCap的节点个数
    // 用于红黑树调整后是否需要在红黑树与二叉树之间转换
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        // 取出该节点的下一个节点
        next = (TreeNode<K,V>)e.next;
        // 旧链表的赋值为空，以便垃圾回收
        e.next = null;
        // 若当前节点的hash与旧table的容量进行与运算结果为零，
        // 则该节点在新table与旧table中的索引位置相同
        if ((e.hash & bit) == 0) {
            // 当前节点的前节点赋值为loTail，若为空，则该节点为根节点
            if ((e.prev = loTail) == null)
                // 根节点赋值为e
                loHead = e;
            else
                // 否则loTail的下个节点赋值为当前节点e
                loTail.next = e;
            // 当前节点赋值给loTail
            loTail = e;
            // 记录与旧节点索引位置相同的节点个数
            ++lc;
        }
        // 若当前节点的hash与旧table的容量进行与运算结果不为零，
        // 则该节点在新table与旧table中的索引位置相同
        else {
            // 当前节点的前节点赋值为hiTail，若为空，则该节点为根节点
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                // 否则hiTail的下个节点赋值为当前节点e
                hiTail.next = e;
            // 当前节点赋值给loTail
            hiTail = e;
            // 记录拆分到新table索引的节点个数
            ++hc;
        }
    }

   // 旧table索引位置节点不为空
    if (loHead != null) {
        // 判断是否小于转换为链表的临界值
        if (lc <= UNTREEIFY_THRESHOLD)
            // 红黑树转链表
            tab[index] = loHead.untreeify(map);
        else {
            // 链表不需要转换，直接插入
            tab[index] = loHead;
            // 新table的索引位置节点不为空(旧table的红黑树已被拆分)，
            // 重新构建红黑树
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    // 新table索引位置节点不为空
    if (hiHead != null) {
        // 判断是否小于转换为链表的临界值
        if (hc <= UNTREEIFY_THRESHOLD)
            // 红黑树转链表
            tab[index + bit] = hiHead.untreeify(map);
        else {
            // 链表不需要转换，直接插入
            tab[index + bit] = hiHead;
            // 旧table的索引位置节点不为空(旧table的红黑树部分拆分到新的位置)，
            // 重新构建红黑树
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}


/**
*  红黑树转链表
*/
final Node<K,V> untreeify(HashMap<K,V> map) {
    // hd：根节点
    Node<K,V> hd = null, tl = null;
    // this：调用此方法的TreeNode
    for (Node<K,V> q = this; q != null; q = q.next) {
        // 新建一个node节点
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            // 记录下一个节点
            tl.next = p;
        tl = p;
    }
    // 返回链表
    return hd;
}


/**
* 构建红黑树
*/
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    // this：调用此方法的TreeNode
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        // 获取下个节点
        next = (TreeNode<K,V>)x.next;
        // 左右孩子清空
        x.left = x.right = null;
        // root为空时，当前节点父节点设为空，颜色变为黑色，赋值给root
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            // 获取当前节点x的key值
            K k = x.key;
            // 获取当前节点x的hash值
            int h = x.hash;
            Class<?> kc = null;
            // 从根节点开始循环，将当前节点变为左孩子或右孩子
            for (TreeNode<K,V> p = root;;) {
                // dir：向左（-1/0）或向右（1），ph：记录当前节点key的hash值
                int dir, ph;
                // 获取当前节点的key
                K pk = p.key;
                // 当前节点的hash大于要查找的节点hash值
                if ((ph = p.hash) > h)
                    // 向左查找
                    dir = -1;
                else if (ph < h)
                    // 向右查找
                    dir = 1;
                // 如果要查找的节点X的key值没有实现compareble接口
                // 或者要查找的节点X的key值与当前节点P的值相等
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 使用自定义的比较规则决定向左或向右查找
                    dir = tieBreakOrder(k, pk);
                // 当前节点P赋值给xp
                TreeNode<K,V> xp = p;
                // 左孩子或右孩子为空时插入
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // 查找节点X的父节点设置为p
                    x.parent = xp;
                    // 判断成为左孩子或右孩子
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 平衡红黑树，主要涉及变色和旋转
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 确保root节点为根节点
    moveRootToFront(tab, root);
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

假设在并发情况下，有两个线程同时扩容，都走到了`transfer()`方法，线程1在执行完` Entry<K,V> next = e.next;`这一行，失去CPU时间片，此时**e指向e0，next指向e1**。

![](..\images\connection\start.png)



而线程2继续执行，线程2扩容完之后，根据缓存一致性协议，新的table最终结果如下图（假设扩容后索引位置未变）：

![](..\images\connection\t2-end.png)

此时线程1重新获取获取CPU执行权限，继续扩容，第一次循环结束，结果如图：

![](..\images\connection\t1-1.png)



next不为空，继续进行第二次循环：

![](D:\JavaNotes\JavaNotes\images\connection\end.png)

**在1.8及之后，扩容时移动数据取消了头插入，解决了死循环的问题**。



### 5. 为什么负载因子要用0.75

在源码中可以这样一段话：

```java
* <p>As a general rule, the default load factor (.75) offers a good tradeoff
* between time and space costs.  Higher values decrease the space overhead
* but increase the lookup cost (reflected in most of the operations of the
* <tt>HashMap</tt> class, including <tt>get</tt> and <tt>put</tt>).  The
* expected number of entries in the map and its load factor should be taken
* into account when setting its initial capacity, so as to minimize the
* number of rehash operations.  If the initial capacity is greater
* than the maximum number of entries divided by the load factor, no
* rehash operations will ever occur. 
```

翻译过来的大概意思就是：

> 负载因子选择0.75是对时间和空间的一种很好的权衡，比较大的值会减少空间开销，但是会增加查找的时间成本。

HashMap中除了哈希算法之外，有两个参数影响了性能：初始容量和加载因子。初始容量是哈希表在创建时的容量，**加载因子是哈希表在其容量自动扩容之前可以达到多满的一种度量**。

在维基百科来描述加载因子：

> 对于开放定址法，加载因子是特别重要因素，**应严格限制在0.7-0.8以下。超过0.8，查表时的CPU缓存不命中（cache missing）按照指数曲线上升**。因此，一些采用开放定址法的hash库，如Java的系统库限制了加载因子为0.75，超过此值将resize散列表。



### 6. 为什么容量要设置为2的n次方

HashMap的底层是数组+链表的结构，对于数组其索引范围是[0, length -1]，为了数据的均匀分布，最直接的办法就是采用取模运算，而在HashMap中采用位运算`hash & (length - 1)`计算索引位置，是因为使用位运算的效率远高于取模运算。

为什么位运算可以代替取模运算呢？实现原理如下：

```
X % 2^n = X & (2^n – 1)
```

假设n为3，则2^3 = 8，表示成2进制就是1000。2^3 -1 = 7 ，即0111。

此时X & (2^3 – 1) 就相当于取X的2进制的最后三位数。

从2进制角度来看，X / 8相当于 X >> 3，即把X右移3位，此时得到了X / 8的商，而被移掉的部分(后三位)，则是X % 8，也就是余数。

所以只要保证length的长度是2^n 的话，就可以实现取模运算了。





## 附录

> #### 红黑树

在某些情况下二叉查找树可能会变成单链表结构(如插入的值一直小于父节点)，从而导致性能大打折扣，红黑树是一种平衡二叉查找树，其最长路径不会超过最短路径的两倍，除了符合二查找叉树的特性外，还具有一下五种特性

1. 每个节点要么是黑色，要么是红色；
2. 根节点是黑色；
3. 每个叶节点是黑色；
4. 红节点的子节点都是黑色；
5. 任意黑色节点到每个子节点的路径都包含相同个数的黑色节点。




## 参考

https://blog.csdn.net/NYfor2017/article/details/105454097

https://juejin.cn/post/6844904016363651085	