# HashMap

[TOC]

## 1.HashMap的长度为什么要设置为2的N次幂

​	`HashMap`底层采用的是数组+链表的结构来存储数据，在jdk1.8开始采用了数组+链表+红黑树的方式来存储。

​	为了实现尽量均匀的Hash，通常采用取模的方式，即index=hash % length，它的映射集合为{0,1...,length-1}，但是取模的方式效率较低，在HashMap中采用了位运算的方式，即index = hash & (length-1)，因为HashMap的长度为2的N次幂，在进行位运算时，index的值取决于hash值的后几位，只要添加的hash本身分布均匀，hash算法得到的index就是均匀的。

​	

##  2. HashMap的属性及内部类

 	在jdk1.7之前，HashMap的采用数组加链表的方式来存储数据，而在jdk1.8中加入了红黑树，当链表的长度大于8时，会将链表转化为红黑树，以提升查找速度。

 	HashMap通过获取插入key的hash值，并与数组的长度进行取模运算`hash & table.length -1`得到插入元素的存放位置，若该位置存在元素时，判断两者hash值以及key，相同时覆盖，不同时插入之后的链表中。

​	HashMap初始化容量大小时，若需要继续添加元素，尽量设置为2的n次方

### 2.1 **HashMap的属性**

```java
// 默认的初始容量(16)
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认的负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 当桶上的元素个数大于该值后，就换转换为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 当桶上的元素个数小于该值后，就换转换为链表
static final int UNTREEIFY_THRESHOLD = 6;

// 转化为红黑树对应的最小table容量
static final int MIN_TREEIFY_CAPACITY = 64;

// 记录结构被修改的次数(fast-fail机制)
transient int modCount;

// 存储元素的数组
transient Node<K,V>[] table;

// entrySet()的缓存
transient Set<Map.Entry<K,V>> entrySet;

// 扩容的临界值
int threshold;

// 负载因子
final float loadFactor;
```

- **loadFactor负载因子 和threshold最大容量**

  `threshold`是数组容量和负载因子对应下的最大容量，`threshold=loadFactor*table.length`，超过这个容量后，`HashMap`就会进行扩容`resize()`

### 2.2 构造方法

```java
// 无参构造
public HashMap() {
    // 设置默认的负载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

// 初始化大小
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 初始化数组容量与负载因子
public HashMap(int initialCapacity, float loadFactor) {
    // 小于零抛错
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // 大于最大值时，取最大值
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // NaN表示未定义的值，在0.0f/0.0f的时候为true
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

// 用于查找大于等于initialCapacity最小的2的幂，即让最高位1后面的全部变成1，再加1之后，即可得到2的N次幂
static final int tableSizeFor(int cap) {
    // 减一是为了保证传入的值已经时2的N次幂时，返回相同的结果
    int n = cap - 1;
    // 每一次无符号右移，最高位“1”后“1”的个数翻倍
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

// 含参构造
 public HashMap(Map<? extends K, ? extends V> m) {
    // 负载因子赋值为默认值 
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}


final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    // 获取传入Map的元素个数
    int s = m.size();
    // 传入的Map有值
    if (s > 0) {
        // 数组table为空，根据传入Map的大小计算出初始 容量
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // 传入的Map元素个数大于扩容的临界值，进行扩容
        else if (s > threshold)
            resize();
        // 循环把值塞进Map中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```



### 2.3 Node和TreeNode

- ####  Node

```java
// 实现了Map.Entry，链表
static class Node<K,V> implements Map.Entry<K,V> {
    // 哈希值
    final int hash;
    // 键
    final K key;
    // 值
    V value;
    // 下一个节点
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
	// 重写了hashCode方法
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
	// 重写equals方法
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}


```

- TreeNode


```java
// 红黑树
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    // 父节点
    TreeNode<K,V> parent;  
    // 左孩子
    TreeNode<K,V> left;
    // 右孩子
    TreeNode<K,V> right;
    // 删除后断开next的链接？？？？？？？？？？？？？？？？？？？？
    TreeNode<K,V> prev;   
    // 判断节点颜色
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

    // 获取根节点
    final TreeNode<K,V> root() {
        // 遍历红黑树，若没有父节点，则是根节点
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }

    // 数组索引位置的值更新为根节点，断开根节点前后节点，若有的话，更新数组索引位置原节点与根节点的关系
    static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
        int n;
        // 根节点不为空，数组不为空且有值
        if (root != null && tab != null && (n = tab.length) > 0) {
            // 通过节点的hash值与桶的长度-1进行与运算，得出在桶中的位置
            int index = (n - 1) & root.hash;
            // 取出index索引上的第一个节点
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
            // 根节点与该节点不相等
            if (root != first) {
                Node<K,V> rn;
                // 把数组index上的索引替换为root
                tab[index] = root;
                // 获取根节点的前一个节点
                TreeNode<K,V> rp = root.prev;
                // 判断root节点的下一个节点是否为空，
                if ((rn = root.next) != null)
                    // 根节点的下个节点的前节点指向根节点的前节点，相当于断开根节点与下个节点的链接
                    ((TreeNode<K,V>)rn).prev = rp;
                // 根节点的前节点不为空
                if (rp != null)
                    // 根节点的后节点指向根节点的前节点的后节点，相当于断开根节点与前节点的关系
                    rp.next = rn;
                if (first != null)
                    // 数组上索引位置上原节点指向根节点
                    first.prev = root;
                // 根节点的下个节点指向数组索引位置原节点
                root.next = first;
                root.prev = null;
            }
            // 保证红黑树结构
            assert checkInvariants(root);
        }
    }
    

    // 根据hash值和key值从根节点开始查找
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        // 获取当前节点
        TreeNode<K,V> p = this;
        do {
            int ph, dir; K pk;
            // 获取左右节点
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            // 当前节点的hash值大于要查找元素的ash值，是左孩子
            if ((ph = p.hash) > h)
                p = pl;
            // 否则是右孩子
            else if (ph < h)
                p = pr;
            // 若k值相等，返回当前值
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p;
            // 左孩子为空，p赋值为右孩子
            else if (pl == null)
                p = pr;
            // 右孩子为空，p赋值为左孩子
            else if (pr == null)
                p = pl;
            // kc不为空或者kc实现了Comparable接口且键k和当前节点不相等
            else if ((kc != null ||
                      (kc = comparableClassFor(k)) != null) &&
                     (dir = compareComparables(kc, k, pk)) != 0)
                // 小于当前节点的健值取左孩子否则取右孩子
                p = (dir < 0) ? pl : pr;
            // 找到值，直接返回
            else if ((q = pr.find(h, k, kc)) != null)
                return q;
            // 以上条件都不满足时，赋值为左孩子
            else
                p = pl;
            
            // 查找到值后，停止循环
        } while (p != null);
        return null;
    }

    // comparableClassFor，x是否实现了Comparable接口，没实现则返回null
    static Class<?> comparableClassFor(Object x) {
        if (x instanceof Comparable) {
            Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
            if ((c = x.getClass()) == String.class) // bypass checks
                return c;
            if ((ts = c.getGenericInterfaces()) != null) {
                for (int i = 0; i < ts.length; ++i) {
                    if (((t = ts[i]) instanceof ParameterizedType) &&
                        ((p = (ParameterizedType)t).getRawType() ==
                         Comparable.class) &&
                        (as = p.getActualTypeArguments()) != null &&
                        as.length == 1 && as[0] == c) // type arg is c
                        return c;
                }
            }
        }
        return null;
    }
    
    // compareComparables： 如果X与kc类型不同返回0，否则用compareTo比较
    // compareTo：-1 小于；0 等于；1 大于
    static int compareComparables(Class<?> kc, Object k, Object x) {
        return (x == null || x.getClass() != kc ? 0 :
                ((Comparable)k).compareTo(x));
    }
    
    /**
     * Calls find for root node.
     */
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);
    }

    /**
     * Tie-breaking utility for ordering insertions when equal
     * hashCodes and non-comparable. We don't require a total
     * order, just a consistent insertion rule to maintain
     * equivalence across rebalancings. Tie-breaking further than
     * necessary simplifies testing a bit.
     */
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
             compareTo(b.getClass().getName())) == 0)
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                 -1 : 1);
        return d;
    }

    /**
     * Forms tree of the nodes linked from this node.
     */
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (root == null) {
                x.parent = null;
                x.red = false;
                root = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                              (kc = comparableClassFor(k)) == null) ||
                             (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);

                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }

    /**
     * Returns a list of non-TreeNodes replacing those linked from
     * this node.
     */
    final Node<K,V> untreeify(HashMap<K,V> map) {
        Node<K,V> hd = null, tl = null;
        for (Node<K,V> q = this; q != null; q = q.next) {
            Node<K,V> p = map.replacementNode(q, null);
            if (tl == null)
                hd = p;
            else
                tl.next = p;
            tl = p;
        }
        return hd;
    }

    /**
     * Tree version of putVal.
     */
    final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                   int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        TreeNode<K,V> root = (parent != null) ? root() : this;
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p;
            else if ((kc == null &&
                      (kc = comparableClassFor(k)) == null) ||
                     (dir = compareComparables(kc, k, pk)) == 0) {
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null &&
                         (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                         (q = ch.find(h, k, kc)) != null))
                        return q;
                }
                dir = tieBreakOrder(k, pk);
            }

            TreeNode<K,V> xp = p;
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                Node<K,V> xpn = xp.next;
                TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                xp.next = x;
                x.parent = x.prev = xp;
                if (xpn != null)
                    ((TreeNode<K,V>)xpn).prev = x;
                moveRootToFront(tab, balanceInsertion(root, x));
                return null;
            }
        }
    }

    /**
     * Removes the given node, that must be present before this call.
     * This is messier than typical red-black deletion code because we
     * cannot swap the contents of an interior node with a leaf
     * successor that is pinned by "next" pointers that are accessible
     * independently during traversal. So instead we swap the tree
     * linkages. If the current tree appears to have too few nodes,
     * the bin is converted back to a plain bin. (The test triggers
     * somewhere between 2 and 6 nodes, depending on tree structure).
     */
    final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                              boolean movable) {
        int n;
        if (tab == null || (n = tab.length) == 0)
            return;
        int index = (n - 1) & hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
        TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
        if (pred == null)
            tab[index] = first = succ;
        else
            pred.next = succ;
        if (succ != null)
            succ.prev = pred;
        if (first == null)
            return;
        if (root.parent != null)
            root = root.root();
        if (root == null
            || (movable
                && (root.right == null
                    || (rl = root.left) == null
                    || rl.left == null))) {
            tab[index] = first.untreeify(map);  // too small
            return;
        }
        TreeNode<K,V> p = this, pl = left, pr = right, replacement;
        if (pl != null && pr != null) {
            TreeNode<K,V> s = pr, sl;
            while ((sl = s.left) != null) // find successor
                s = sl;
            boolean c = s.red; s.red = p.red; p.red = c; // swap colors
            TreeNode<K,V> sr = s.right;
            TreeNode<K,V> pp = p.parent;
            if (s == pr) { // p was s's direct parent
                p.parent = s;
                s.right = p;
            }
            else {
                TreeNode<K,V> sp = s.parent;
                if ((p.parent = sp) != null) {
                    if (s == sp.left)
                        sp.left = p;
                    else
                        sp.right = p;
                }
                if ((s.right = pr) != null)
                    pr.parent = s;
            }
            p.left = null;
            if ((p.right = sr) != null)
                sr.parent = p;
            if ((s.left = pl) != null)
                pl.parent = s;
            if ((s.parent = pp) == null)
                root = s;
            else if (p == pp.left)
                pp.left = s;
            else
                pp.right = s;
            if (sr != null)
                replacement = sr;
            else
                replacement = p;
        }
        else if (pl != null)
            replacement = pl;
        else if (pr != null)
            replacement = pr;
        else
            replacement = p;
        if (replacement != p) {
            TreeNode<K,V> pp = replacement.parent = p.parent;
            if (pp == null)
                root = replacement;
            else if (p == pp.left)
                pp.left = replacement;
            else
                pp.right = replacement;
            p.left = p.right = p.parent = null;
        }

        TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

        if (replacement == p) {  // detach
            TreeNode<K,V> pp = p.parent;
            p.parent = null;
            if (pp != null) {
                if (p == pp.left)
                    pp.left = null;
                else if (p == pp.right)
                    pp.right = null;
            }
        }
        if (movable)
            moveRootToFront(tab, r);
    }

 

    /* ------------------------------------------------------------ */
    // Red-black tree methods, all adapted from CLR

    static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                          TreeNode<K,V> p) {
        TreeNode<K,V> r, pp, rl;
        if (p != null && (r = p.right) != null) {
            if ((rl = p.right = r.left) != null)
                rl.parent = p;
            if ((pp = r.parent = p.parent) == null)
                (root = r).red = false;
            else if (pp.left == p)
                pp.left = r;
            else
                pp.right = r;
            r.left = p;
            p.parent = r;
        }
        return root;
    }

    static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                           TreeNode<K,V> p) {
        TreeNode<K,V> l, pp, lr;
        if (p != null && (l = p.left) != null) {
            if ((lr = p.left = l.right) != null)
                lr.parent = p;
            if ((pp = l.parent = p.parent) == null)
                (root = l).red = false;
            else if (pp.right == p)
                pp.right = l;
            else
                pp.left = l;
            l.right = p;
            p.parent = l;
        }
        return root;
    }

    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                TreeNode<K,V> x) {
        x.red = true;
        for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
            if ((xp = x.parent) == null) {
                x.red = false;
                return x;
            }
            else if (!xp.red || (xpp = xp.parent) == null)
                return root;
            if (xp == (xppl = xpp.left)) {
                if ((xppr = xpp.right) != null && xppr.red) {
                    xppr.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.right) {
                        root = rotateLeft(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateRight(root, xpp);
                        }
                    }
                }
            }
            else {
                if (xppl != null && xppl.red) {
                    xppl.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.left) {
                        root = rotateRight(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateLeft(root, xpp);
                        }
                    }
                }
            }
        }
    }

    static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                               TreeNode<K,V> x) {
        for (TreeNode<K,V> xp, xpl, xpr;;) {
            if (x == null || x == root)
                return root;
            else if ((xp = x.parent) == null) {
                x.red = false;
                return x;
            }
            else if (x.red) {
                x.red = false;
                return root;
            }
            else if ((xpl = xp.left) == x) {
                if ((xpr = xp.right) != null && xpr.red) {
                    xpr.red = false;
                    xp.red = true;
                    root = rotateLeft(root, xp);
                    xpr = (xp = x.parent) == null ? null : xp.right;
                }
                if (xpr == null)
                    x = xp;
                else {
                    TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                    if ((sr == null || !sr.red) &&
                        (sl == null || !sl.red)) {
                        xpr.red = true;
                        x = xp;
                    }
                    else {
                        if (sr == null || !sr.red) {
                            if (sl != null)
                                sl.red = false;
                            xpr.red = true;
                            root = rotateRight(root, xpr);
                            xpr = (xp = x.parent) == null ?
                                null : xp.right;
                        }
                        if (xpr != null) {
                            xpr.red = (xp == null) ? false : xp.red;
                            if ((sr = xpr.right) != null)
                                sr.red = false;
                        }
                        if (xp != null) {
                            xp.red = false;
                            root = rotateLeft(root, xp);
                        }
                        x = root;
                    }
                }
            }
            else { // symmetric
                if (xpl != null && xpl.red) {
                    xpl.red = false;
                    xp.red = true;
                    root = rotateRight(root, xp);
                    xpl = (xp = x.parent) == null ? null : xp.left;
                }
                if (xpl == null)
                    x = xp;
                else {
                    TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                    if ((sl == null || !sl.red) &&
                        (sr == null || !sr.red)) {
                        xpl.red = true;
                        x = xp;
                    }
                    else {
                        if (sl == null || !sl.red) {
                            if (sr != null)
                                sr.red = false;
                            xpl.red = true;
                            root = rotateLeft(root, xpl);
                            xpl = (xp = x.parent) == null ?
                                null : xp.left;
                        }
                        if (xpl != null) {
                            xpl.red = (xp == null) ? false : xp.red;
                            if ((sl = xpl.left) != null)
                                sl.red = false;
                        }
                        if (xp != null) {
                            xp.red = false;
                            root = rotateRight(root, xp);
                        }
                        x = root;
                    }
                }
            }
        }
    }

    // 递归检查
    static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
        // 取出该节点的父节点，左右孩子，前节点，后节点
        TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
            tb = t.prev, tn = (TreeNode<K,V>)t.next;
        if (tb != null && tb.next != t)
            return false;
        if (tn != null && tn.prev != t)
            return false;
        if (tp != null && t != tp.left && t != tp.right)
            return false;
        if (tl != null && (tl.parent != t || tl.hash > t.hash))
            return false;
        if (tr != null && (tr.parent != t || tr.hash < t.hash))
            return false;
        if (t.red && tl != null && tl.red && tr != null && tr.red)
            return false;
        if (tl != null && !checkInvariants(tl))
            return false;
        if (tr != null && !checkInvariants(tr))
            return false;
        return true;
    }
}
```



## 3 添加方法 put(K key, V value)

1.  判断数组是否已经初始化且长度是否为零，若不满足，则进行扩容/初始化，resize()
2.  计算插入的索引位置，若该位置没有值，直接插入，若key相同，则覆盖更新
3.  该索引位置的节点为红黑树节点，则对红黑树进行分割
4.  循环索引位置的链表，将value更新进去
5.  插入值后判断是否需要扩容

```java
// 添加key-value，并返回该位置的原值，没有则返回空
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 获取hash值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
/**
 * @param hash key的hash值
 * @param key key的值
 * @param value value的值
 * @param onlyIfAbsent 是否替换现在额值
 * @param evict if false, the table is in creation mode.
 * @return previous 返回值
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // tab：临时记录table桶，p：临时记录node，n：临时揭露table的长度，i：临时记录索引值
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1. 桶未初始化或者长度为0时进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2. 若该索引位置为空，则新建Node节点，插入该位置，若key相同则覆盖
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 该位置有值，且key值相同，进行覆盖更新
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 3. 该位置有值且为红黑树节点，则对该红黑树进行分割处理
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
             // 4. 循环改table索引位置链表
            for (int binCount = 0; ; ++binCount) {
                // 下个节点为空，新建节点直接插入，终止循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 下个节点不为空，且key值相同，覆盖更新，终止循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 当前节点的下个节点赋值给p
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
    // 5. 判断是否需要扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```

#### 扩容/初始化数组方法：resize()

```java
// 扩容方法
final Node<K,V>[] resize() {
    // 获取数组
    Node<K,V>[] oldTab = table;
    // 获取数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 扩容临界值
    int oldThr = threshold;
    // newCap: 扩容后新的长度，newThr: 新的扩容临界值
    int newCap, newThr = 0;
    // 数组长度大于零
    if (oldCap > 0) {
        // 长度大于等于最大容量时，扩容临界值取Integer最大值，直接返回
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容两倍后，小于最大容量，且大于初始化容量时，扩容两倍，记录扩容后的容量和扩容临界值
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 数组为空，初始容量设置为扩容临界值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {              
        // 初始化时使用的无参构造，长度以及扩容临界值取默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 新的扩容临界值为零(即数组为空的情况)，计算新的扩容临界值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 给HashMap的扩容临界值重新赋值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 根据新的长度，新建数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 旧数组重新赋值
    table = newTab;
    // 旧数组不为空
    if (oldTab != null) {
        // 循环将旧数组中的值赋值到扩容后的新数组中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 值不为空
            if ((e = oldTab[j]) != null) {
                // 清空旧数组索引位置的值
                oldTab[j] = null;
                // 索引位置只有一个值，把值存入新位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 该节点是红黑树节点
                else if (e instanceof TreeNode)
                    // TreeNode的内部方法，数组该索引位置后的红黑树
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
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
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```

####  红黑树的分割方法：split()

```java

/**
*  ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
* 只在resize()方法中调用，在table扩容后，判断红黑树是否转为链表
* 计算红黑树中的节点在新数组中的索引位置，并插入到该位置
*
* tab: 扩容或初始化后新的table数组
* index：在数组的索引位置，
* bit/oldCap: 旧数组长度
*/
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // 临时存储与旧table索引位置相同的节点
    TreeNode<K,V> loHead = null, loTail = null;
    // 临时存储与旧table索引+oldCap的节点
    TreeNode<K,V> hiHead = null, hiTail = null;
    // lc：记录与旧table中索引相同的节点个数，hc：记录旧table+oldCap的节点个数
    // 原来判断是否需要在红黑树与二叉树之间转换
    int lc = 0, hc = 0;
    // 遍历index位置的红黑树，
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
    // ------------- 循环结束------------//
    
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

// 红黑树转链表
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

// 新建node节点
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    return new Node<>(p.hash, p.key, p.value, next);
}


// 构建红黑树
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
                    // 平衡红黑树
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 确保root节点为根节点
    moveRootToFront(tab, root);
}

// 比较key值
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}

static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        (d = a.getClass().getName().
         compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
             -1 : 1);
    return d;
}
```



#### 红黑树插入节点时的调整

```java
// 1. 变色  
// 2. 旋转
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    // 把x节点设置为红色节点
    x.red = true;
    // 自下而上进行调整
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        // x的父节点xp为空，把x节点设置为黑色终止循环
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        // xp为黑色节点，或者xp的父节点为空，返回root节点
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        if (xp == (xppl = xpp.left)) {
            // xp与父节点的左孩子相等，且右孩子为红色节点时
            // xp父节点的右孩子与xp变为黑色，xp的父节点变为红色，xp的父节点赋值为x节点
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            // xp
            else {
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```





## 附件

### 1.  位运算符

| 操作符 | 描述                                      |
| :----- | :---------------------------------------- |
| &      | 如果相对应位置都是1，则结果为1，否则为0   |
| \|     | 如果相对应位置都是0，则结果为0，否则都是1 |
| ^      | 如果对应位置相同，则结果为1，否则为0      |



### 2. 取余(rem)和取模(mod)

对与整数x, y取余和取模都需要两步：

1.  z = x/y
2.  r = x-(z*y)

取余和取模运算在第一步不同，取余运算商的值会趋向于0，而取模运算商的值会趋向于负无穷；

例如 5/(-3) 约等于 -1.6，取余运算得到的商为-1，而取模运算得到的商为-2。

在java中 % 运算得到的余数与被除数符号相同



### 3. 红黑树

二叉查找树(BST)的特性

1. 左节点的值小于或等于父节点
2. 右节点的值大于或等于父节点

在某些情况下二叉查找树可能会变成单链表结构(如插入的值一直小于父节点)，从而导致性能大打折扣，红黑树是一种平衡二叉查找树，其最长路径不会超过最短路径的两倍，除了符合二查找叉树的特性外，还具有一下五种特性

1. 每个节点要么是黑色，要么是红色
2. 根节点是黑色
3. 每个叶节点是黑色
4. 红节点的子节点都是黑色
5. 任意黑色节点到每个子节点的路径都包含相同个数的黑色节点

当插入或删除元素破坏红黑树的平衡后，会采用**变色**和**旋转**来调整。

**变色：**为了使红黑树重新符合规则，尝试把红色节点变成黑色节点后者把黑色节点变成红色节点；

**旋转：**分为左旋转和右旋转。

**左旋转**：右孩子取代父节点，且右孩子的左节点变为父节点的右节点

![](D:\笔记\图片\hashmap.png)

​	

**右旋转**：左孩子取代父节点，且左孩子的右节点变为父节点的左节点 

![](D:\笔记\图片\hashmap2.png)