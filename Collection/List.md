# List

[TOC]



## 1. LinkedList和ArrayList的区别

- **底层结构**：`ArrayList`底层是`Object`数组，`LinkedList`底层是双向链表

- **是否安全**：`ArrayList`和`LinkedList`都不是同步的，线程不安全

- **增删和查找**：`ArrayList`使用数组存储，增加和删除元素的时间复杂度受到位置的影响，`add(E e)`方法默认

  追加到列表的末尾，此时时间复杂度为O(1)，在指定位置 i 添加或删除元素时，时间复杂度为O(n-i)，查找时可以通过索引实现快速查找，`LinkedList`使用链表存储，在增加和删除时不受位置的影响时间复杂度近似O(1)，但其无法进行快速查找`get(int i)`是通过循环遍历来查找

  

## 2. ArrayList源码



### 2.1 构造方法

​	`RandcomAccess`接口，可以看作一个标志接口，实现该接口标志着是支持快速访问的

​	`Cloneable`接口，实现该接口标志着可以被克隆复制

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
   private static final long serialVersionUID = 8683452581122892189L;
    // 默认初始容量
   private static final int DEFAULT_CAPACITY = 10;
    
    // 有参构造实例化数组的初始化值
   private static final Object[] EMPTY_ELEMENTDATA = {};

    // 无参构造实例化数组的初始化值
   private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 存储数据的数组
   transient Object[] elementData; // non-private to simplify nested class access
   
   // 数组的默认最大容量，一些虚拟机在数组中保留了头信息
   // 最大容量为Integer.MAX_VALUE
   private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
   
	// 记录ArrayList被修改次数,添加
   protected transient int modCount = 0;

    // 数组中元素个数
   private int size;
    
   public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    
    // 指定初始大小
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    // 默认构造方法，当添加第一个元素时容量扩充到默认大小10
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    public ArrayList(Collection<? extends E> c) {
    	// 转化为数组
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // 有可能转化的不是Object类型的数组
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 长度为空时，赋值为EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
}
```



### 2.2  添加方法

​	`ArrayList`在添加元素造成扩容时，使用到了两个方法，分别时`Arrays.copyOf()`和`System.arraycopy`两个方法。`Arrays.copyOf`最终调用的也是`System.arraycopy`，进行扩容后，会返回一个新的数组。

- #### add(E e)  插入一个元素

```java
// 在结尾添加元素，判断是否需要扩容（扩容大小为原大小1.5倍），若需要将原数组复制进扩容后的数组
public boolean add(E e) {
    // 判断是否需要扩容，若需要则将旧数组复制到新数组内
    ensureCapacityInternal(size + 1);
    // 将新增的元素放在数组最后
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 是否是用无参构造实例化的ArrayList
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 返回默认大小与新元素个数最大值
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // add()导致超出现容量，进行扩容
    if (minCapacity - elementData.length > 0)
       grow(minCapacity);
}

private void grow(int minCapacity) {
	int oldCapacity = elementData.length;
    // 新长度为旧长度的1.5倍
	int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 新增元素后的长度是否大于newCapacity（addAll()方法可能会产生大于的情况）
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 新长度超过数组最大容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
        // 将旧数组的值复制到扩容后的新数组，并返回新数组
        elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 现容量+1后大于数组默认最大容量，扩容为Integer.MAX_VALUE，否则扩容为默认最大容量
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

- #### add(int index, E e) 在指定位置插入元素

```java
public void add(int index, E element) {
    // 校验索引index是否超出ArrayList元素个数或小于零
    rangeCheckForAdd(index);
    // 与add(E e)方法相同
    ensureCapacityInternal(size + 1); 
    // 从index开始复制自己
    // System.arraycopy(源数组，源数组起始位置(包含index)，目标数组，起始位置(包含index),复制长度)
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    // 替换索引index上的值
    elementData[index] = element;
    size++;
}
```

- #### addAll(Collection<? Extends E> c)  在结尾插入集合

```java
public boolean addAll(Collection<? extends E> c) {
    // 把集合转化为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    // 同add(E e)
    ensureCapacityInternal(size + numNew);
    // 将新增的数组复制到原数组中
    System.arraycopy(a, 0, elementData, size, numNew);
    // 更新元素个数
    size += numNew;
    return numNew != 0;
}
```

- #### addAll(int index, Collection<? Extends E> c)  在指定位置插入集合

```java
public boolean addAll(int index, Collection<? extends E> c) {
    // 校验索引
    rangeCheckForAdd(index);
	// 与addAll(Collection<? Extends E> c)方法相同
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
	// 所需移动元素个数
    int numMoved = size - index;
    // 不是放在最后
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```



### 2.3 其他方法

```java
// 获取元素个数
public int size() {
    return size;
}

// 是否为空
public boolean isEmpty() {
    return size == 0;
}

// 是否包含某个元素
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}

// 元素在数组中的索引
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

// 数组中元素最后一次出现的索引位置
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

// 复制该数组，清零modCount
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}

// ArrayList转化为数组，返回的是新的数组
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

// ArrayList转化为的数组
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // 长度小于List时，返回新的数组
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}

// 获取指定位置索引
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

// 替换索引上的元素
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}

// 删除所有元素
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}

// 删除指定位置上的元素
public E remove(int index) {
    // 检验索引
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    // 移动元素个数
    int numMoved = size - index - 1;
    // 是否是最后一个
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 赋值为空，以便进行回收
    elementData[--size] = null; // clear to let GC do its work
	// 返回该位置上的旧值
    return oldValue;
}

// 删除包含的元素
public boolean remove(Object o) {
    // 删除的元素是否为空，以便使用不同的方法来进行比较
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

// 删除指定集合中不包含的元素
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}

private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        // 从0索引开始替换相同的元素
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
         // 遍历时出错，未遍历完，将位遍历的元素复制到后面
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        // 包含元素数量不相同，剩余的元素设置为空
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}

//  是否包含指定集合中的所有元素
public boolean containsAll(Collection<?> c) {
    for (Object e : c)
        if (!contains(e))
            return false;
    return true;
}

// 从指定位置获取迭代器
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

// 获取迭代器
public ListIterator<E> listIterator() {
    return new ListItr(0);
}

// 获取迭代器
public Iterator<E> iterator() {
    return new Itr();
}

```



### 2.4 内部类

- #### Itr：

  重写了`hasNext()，next(), remove()`方法，可以在迭代的时候删除元素

  Itr的next()方法，每调用一次，`cursor`都会加一，同时把值赋给`lastRet`。

```java
private class Itr implements Iterator<E> {
    // 下一个元素的索引
    int cursor;       // index of next element to return
    // 最后一个元素的索引
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}
	
    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        // 检查expectedModCount和modCount是否相等
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

- #### ListItr

  继承`Itr`,重写了`hasPrevious(), nextIndex(), previousIndex(), set(E e), add(E e)`方法，可以在迭代的时候增加元素

```java
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }

    public boolean hasPrevious() {
        return cursor != 0;
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor - 1;
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```



## 3. LinkedList 源码

​	`LinkedList`的数据结构为双链表，其实现了Deque接口(继承与Queune)，具有了队列的性质。  

![](D:\笔记\图片\LinkedList.png)

​	`LinkedList`底层采用Node<E>节点来存储信息

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```



### 3.1 关于队列的一些知识

​	队列(Quenue)，是一种先进先出(FIFO)的数据结构，队列只允许在头部删除，尾部插入。

```java
public interface Queue<E> extends Collection<E> {
    // 添加元素，在超出容量时抛出异常
    boolean add(E e);

    // 添加元素，在超出容量时返回false,不抛出异常
    boolean offer(E e);

    // 删除队列头元素并返回，队列为空(元素个数为0，并非队列为null)时抛出异常
    E remove();

    // 删除队列头元素并返回，队列为空返回null
    E poll();

     // 返回列头元素，队列为空时抛出异常
    E element();

    // 返回列头元素，队列为空返回null
    E peek();
}

```



### 3.2 构造方法

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
	transient int size = 0;

    // 指向第一个节点
    transient Node<E> first;

    // 指向最后一个节点
    transient Node<E> last;

    // 无参构造
    public LinkedList() {
    }

    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
}
```



### 3.3 添加方法

- #### add(E e) /offer(E e) 在结尾添加元素

```java
// 新建一个节点作为最后一个节点，并将前一个节点指向原最后节点，原最后节点指向新节点
public boolean add(E e) {
    // addLast(E e) 也是调用的linkLast(e)
    linkLast(e);
    return true;
}

void linkLast(E e) {
    // 把最后的节点赋值给下一个Node的prev
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    // 把新实例化的Node赋值给指向最后的节点
    last = newNode;
    // 如果是新增的第一个节点，赋值给指向第一的节点
    if (l == null)
        first = newNode;
    else
        // 前一个结点的next指向下一个节点
        l.next = newNode;
    size++;
    modCount++;
}

public boolean offer(E e) {
    return add(e);
}
```

- #### add(int index, E element) 在指定位置添加元素

```java
// 利用一次二分，找到索引位置的元素，更新插入元素与前后元素的关系
public void add(int index, E element) {
    // 校验index是否小于零或者大于元素个数
    checkPositionIndex(index);
	
    // 添加的最后
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

// LinkedList无法像ArrayList直接通过get(index)取值，所以需要遍历
Node<E> node(int index) {
    // 如果插入位置小于长度一半，取出索引位置的元素
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 如果插入位置大于长度一半，取出索引位置的元素
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}


void linkBefore(E e, Node<E> succ) {
    // 获取旧节点前一个节点
    final Node<E> pred = succ.prev;
    // 在索引位置创建一个新节点,前节点为该位置原前节点，后节点为该位置旧节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 该位置旧节点的前一个节点更新为新节点
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
       // 该位置前节点的后节点更新新节点
        pred.next = newNode;
    size++;
    modCount++;
}
```

- ####  addFirst/offerFirst(E e)/push(E e)：在头添加元素

```java
public void addLast(E e) {
    linkLast(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    // 新节点指向原第一个节点
    final Node<E> newNode = new Node<>(null, e, f);
    // 新节点变成第一个节点
    first = newNode;
    if (f == null)
        last = newNode;
    else
        // 第一个节点指向新的第一个节点
        f.prev = newNode;
    size++;
    modCount++;
}

public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

public void push(E e) {
    addFirst(e);
}
```

- #### addAll(Collection<? extends E> ) 在最后插入集合

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

// 插入位置是最后一个，循环
public boolean addAll(int index, Collection<? extends E> c) {
    // 校验索引是否小于零或者大于现元素个数
    checkPositionIndex(index);
    // 集合转换为数组，若为空，直接返回
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;
	// pred：在循环插入值时，作为每个新建节点的前一个元素
    Node<E> pred, succ;
    // 拆入位置在LinkedList的最后
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        // 位置不是最后，取出该索引位置的元素
        succ = node(index);
        // 该索引位置的上一个节点赋值给pred
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        // 新建一个元素，上一个元素为索引位置的上一个
        Node<E> newNode = new Node<>(pred, e, null);
        // LinkedList为空，新建的节点赋值第一个
        if (pred == null)
            first = newNode;
        else
            // 索引位置前一节点的后节点更新为新建的节点
            pred.next = newNode;
        // 更新pred的值，用于下次循环作为前一个节点使用
        pred = newNode;
    }
	
    // 插入索引位置在最后
    if (succ == null) {
        last = pred;
    } else {
        // 索引位置节点后移
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```



### 3.4 查找方法

- #### indexOf(Object o) 获取某个元素在集合中的位置

```java
public int indexOf(Object o) {
    int index = 0;
    // 元素为空，循环时用==判断
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        // 元素不为空用equals判断
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

- #### contains(Object o) 判断LinkedList中是否存在某个元素

```java
public boolean contains(Object o) {
    return indexOf(o) != -1;
}
```

- ### get(int index) 获取指定位置的元素

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    // 小于元素个数一半，从头寻找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 大于一半，从尾开始找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

- #### peek()  获取（队列）第一个元素

```java
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```



### 3.5 删除方法

