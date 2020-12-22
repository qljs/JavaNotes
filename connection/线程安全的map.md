



## 一 Jdk1.7 中的ConcurrentHashMap



### 1. 底层结构

在1.7版本中ConcurrentHashMap底层有Segment[] + 数组 + 链表构成，每个 Segment中包含多个HashEntry数组。

Segment的长度一旦初始化后就不能修改，默认是16。

![](..\images\connection\conhashmap1.7.png)



### 2. 构造函数

```java
public ConcurrentHashMap() {
    // 初始容量（DEFAULT_INITIAL_CAPACITY）：16
    // 默认负载因子（DEFAULT_LOAD_FACTOR）：0.75
    // 默认并发级别（DEFAULT_CONCURRENCY_LEVEL）：16
	this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}


public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    // 初始化参数的校验
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 校验并发级别，最大时MAX_SEGMENTS（1 << 16）
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // 2的多少次方
    int sshift = 0;
    // 2的sshift次方
    int ssize = 1;
    // 获取最接近concurrencyLevel的2的n次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    // 计算初始容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 计算每个segment的容量
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // MIN_SEGMENT_TABLE_CAPACITY：2
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    // 保证容量为2的整倍数
    while (cap < c)
        cap <<= 1;
    // 创建第一个Segment
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // native方法
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```



### 3. put()

```java
public V put(K key, V value) {
    Segment<K,V> s;
    // 不允许放空值
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    // segmentShift(28)，segmentMask(15)初始化时，在静态代码块完成赋值操作
    // hash值无符号位右移28位，然后和segmentMask做位运算
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // 查找到的Segment为空，初始化
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}


private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    // 索引为u的位置是否为空
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 获取0位置的Segment，构造方法中放入的
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        // 获取HashEntry的初始化长度
        int cap = proto.table.length;
        // 获取负载因子，所有segment中是一样的
        float lf = proto.loadFactor;
        // 计算扩容阈值
        int threshold = (int)(cap * lf);
        // 构建HashEntry数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        // 再次尝试获取，看看是否有其他线程已经初始化了
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 死循环，不断进行CAS操作，直至成功
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}

/**
* 真正的put()方法
*/
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 若没有获取到ReentrantLock锁，通过scanAndLockForPut()获取。
    // scanAndLockForPut()：该方法通过自旋不断尝试获取锁（tryLock），拿到锁后会返回头节点，没拿到若超过一定次数，则阻塞，
    HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        // 获取table上index位置的节点
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            // 当前位置不为空，这个分支只处理存在key相同的情况
            if (e != null) {
                K k;
                // key相同，替换
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 不断向后找
                e = e.next;
            }
            else {
                // 当前节点为空，可能时头节点为空，也可能时上一步链表中没有相同key值的情况
                if (node != null)
                    // 通过scanAndLockForPut()获取锁的，用头插入插入节点
                    node.setNext(first);
                else
                    // node为空，即头节点为空，新建节点
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 判断是否需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    // 调用unsafe的putOrderedObject()将节点插入table
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 释放锁
        unlock();
    }
    return oldValue;
}
```

在1.7中的put()方法大概流程：

1. 计算出在segment[]中的位置，获取该位置的segment，检查是否需要初始化；
2. 调用segment.put()方法，先尝试获取锁，没获取到的，通过scanAndLockForPut()，自旋获取并返回当前位置头节点；
3. 拿到锁的，先获取segment中的HashEntry[]，然后计算在HashEntry[]的索引位置；
4. 当前位置不为空，检查key是否相同，相同则替换，不同插入。



### 4. get()

```java
/**
* get的大概流程也是，先定位到Segment，然后在定位到HashEntry，之后获取值
*/
public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null && (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
             (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```







## 二 Jdk1.8中的ConcurrentHashMap

jdk1.8中的ConcurrentHashMap和HashMap的结构相同，放弃了1.7中分段锁的方式，只是通过CAS、synchronized、volatile来保证线程安全，但是在1.8中仍保留了segment来兼容1.7。

jdk1.8中几个重要的属性：

```java
// 存储数据table，使用volatile修饰，保证了在get时，线程看到的是最新的
transient volatile Node<K,V>[] table;
// 扩容时临时存储数据的table
private transient volatile Node<K,V>[] nextTable;
// 标识数据
// -1:表示正在初始化
// -n:当前有n-1个线程在进行扩容，
// 0:未初始化
// 大于0：下一次扩容的大小
private transient volatile int sizeCtl;
```



### 1. put()

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    // 记录链表长度，用于判断插入后是否需要转化为红黑树
    int binCount = 0;
    // 不断循环整个数组，保证每个线程都可以put成功，
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // table未初始化，进行初始化
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
           // 当前位置没有值，通过CAS尝试插入，成功则返回
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 当前map正在扩容，帮助转移数据
        // 在1.8中对于值为负数的hash进行了特殊处理，以下针对的头节点
        // static final int MOVED     = -1; // 正在扩容
    	// static final int TREEBIN   = -2; // 表示当前是TreeBin节点，用于红黑树的头节点
    	// static final int RESERVED  = -3; // 临时的hash
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 当前节点不为空，锁住头节点
            synchronized (f) {
                // 检查下头节点有没有改变，可能经历了扩容，当前位置的头节点变掉了
                if (tabAt(tab, i) == f) {
                    // 当前为链表
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // key相同，根据onlyIfAbsent选择是否覆盖
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // key不同，插入链表后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树插入操作
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 链表长度超过阈值，转化为红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```



> #### 初始化：initTable()

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 循环判断table是否已初始化
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl表明正在初始化或者扩容
        if ((sc = sizeCtl) < 0)
            // 当前线程让出资源，进入阻塞
            Thread.yield(); // lost initialization race; just spin
        // CAS操作将sizeCtl设置为-1
        // SIZECTL：sizeCtl的偏移量
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 再次判断下table是否为空
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下次扩容1.5倍
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

