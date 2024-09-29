---
title: HasMap底层分析
date: 2024-9-21 11:00:00 +0800
categories: [Java基础]
tags: [java]
---

## 概述

*HashMap*实现了*Map*接口（Map<K, V>），允许存放 key 为 null 的元素，同时也允许插入 value 为 null 的元素；同时实现了*Cloneable*和*Serializable*允许克隆和序列化。

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    transient Node<K,V>[] table;
}
```

*HashMap*的存储结构为**数组+链表+红黑树**（散列表结构），其需要对存入的 key 进行哈希（索引数组下标）来决定存储位置（称为桶 bucket），因此该容器中存储的元素是无序的。当不同的 key 经过哈希后索引到了相同的数组下标，称之为**哈希冲突**，根据对冲突的处理方式不同，哈希表有两种实现方式，一种开放地址方式(Open addressing)，另一种是冲突链表方式(Spearate chaining with linked lists)。

**Java8 HashMap采用的是链式冲突方式**，即链表。

![image-1](/assets/blog-image/blog2/hash1.png)

Node的结构如下：

```java
    /**
     * Basic hash bin node, used for most entries. (See below for TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // key的哈希值
        final int hash;
        // 存储键
        final K key;
        // 存储值
        V value;
        // 指向下一个节点的引用
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

        // 计算节点的哈希值
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

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

Node实现了`Map.Entry<K, V>`，其中存储了四个属性，并定义了哈希值的计算方法：`hashcode()`，同时定义`equals()`方法，比较两个节点的键和值是否相等。

从*HashMap*的结构可以发现，如果选择的哈希函数合适，`put()`和`get()`方法的时间复杂度可以到`O(1)`。

在对*HashMap*进行遍历时，需要遍历整个数组以及后面跟着的冲突链表。因此在需要频繁遍历的场景下，*HashMap*的容量不宜过大。相对应的，链表在增加到一定程度后会转换为**红黑树**，以此来降低遍历链表的开销。

有两个参数可以影响*HashMap*的性能：

- 初始容量(inital capacity)：`Node<K,V>[] table`的初始大小
- 负载系数(load factor)：`float loadFactor`，用于计算自动扩容临界值。

```java
	// 限制最大容量和判断负载系数的合法性
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

	// 指定table大小，默认负载系数为0.75f
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    // 初始
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    // 通过集合创建HashMap
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

*HashMap*定义了四种构造方法，当采用创建*HashMap*实例时，默认`table`为空，`loadFactor`为0.75，在第一次向*HashMap*中插入元素时，会在内部调用`resize()`方法，这时`table`会被初始化为长度16的数组或者自定义的初始容量。

> 通过引用的方式访问 `HashMap` 的 `table` 数组。这样可以在代码中方便地操作哈希表的元素，同时便于在扩容等情况下使用旧的表结构进行重新分配。

```java
        // 引用 table 为 oldTab，避免直接操作成员变量
		Node<K,V>[] oldTab = table;
		// 获取哈希表当前容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
		// 当前扩容阈值
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 扩容阈值翻倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 这里扩容 0 --> 16
            newCap = DEFAULT_INITIAL_CAPACITY;
            // 扩容阈值为 0.75*16=12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
		// 初始化扩容阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
```

## put()过程分析

*HashMap*使用 put 的方式进行数据存储，其中有两个参数，分别是 key 和 value。

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    // onlyIfAbsent如果为true，那么只有在不存在该key时才会进行put操作
     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // tab用于引用访问table
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // table为空，第一次插入时触发resize()进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // hash值与数组长度n-1进行与运算，找到具体的数组下标
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 为空则直接插入
            tab[i] = newNode(hash, key, value, null);
        else { // 数组该位置有数据
            Node<K,V> e; K k;
            // 判断该位置的数据与要插入的数据，key是否”相等“，是则取出该数据
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 判断该节点是否为红黑树的节点，是则调用红黑树的插值方法
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 若进行到此位置，说明数组该位置上有一个链表
                for (int binCount = 0; ; ++binCount) {
                    // 链表的尾插法
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // TREEIFY_THRESHOLD为8，所以，如果新插入的值是链表中的第 8 个
                        // 那么会将链表转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 若链表节点中有”相等“的key（==或equals）
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 跳出循环，那么 e 为链表中[与要插入的新值的 key "相等"]的 node
                        break;
                    p = e;
                }
            }
            // e不为空，说明存在节点的key与要插入的key”相等“
            if (e != null) { // 进行覆盖操作，并返回旧值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 若插入新值后，size超过了阈值，则进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }   
```

对于JDK1.8引入的红黑树结构，会在链表过长时进行转换，那么为什么会以==TREEIFY_THRESHOLD = 8==作为转换的阈值呢？

这里引用官方的说明来解释：

```
* bin中的节点遵循泊松分布
* (http://en.wikipedia.org/wiki/Poisson_distribution) 带有
* 默认调整大小的参数平均约为 0.5
* 阈值为 0.75，尽管由于以下原因存在较大方差
* 调整粒度。 忽略方差，预期
* 列表大小 k 的出现次数为 (exp(-0.5) * pow(0.5, k) /
*阶乘(k))。 第一个值是：
*
* 0: 0.60653066
* 1：0.30326533
* 2：0.07581633
* 3：0.01263606
* 4：0.00157952
* 5：0.00015795
* 6：0.00001316
* 7：0.00000094
* 8：0.00000006
```

选取8是因为发生哈希冲突的概率服从泊松分布，而对于同一个数组下标而言，发生8次哈希冲突的概率极小，此时是一个比较合适的值。

那这里同时又衍生出一个问题，既然有**树化操作**，也会有**链化操作**吧？

答案是肯定的。

在*HashMap*的`remove()`方法中，若移除的节点为`TreeNode`，则会调用其中的`removeTreeNode()`方法，其中转换方法`untreeify(HashMap<K,V> map)`，这里判断的是当树的节点小于==6==时，进行链化。

为什么树化时8，链化就变为6了呢？

是为了防止链表长度在7、8之间来回，从而导致不断链化、树化，浪费资源。

## hash(Object key)

在put过程中，int hash是通过hash()方法计算的，在确定数组下标时，还需要进行`(n - 1) & hash`。

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

先获取 key的hashCode值，将hashCode二进制的高 16 位与低 16 位的进行**异或运算**，到的结果再与当前数组长度减一进行与运算。最终得到一个数组下标。

**int hashCode = key.hashCode()**

**int hash = hash(key) = key.hashCode()的高 16 位^低 16 位 &(n-1) 其中 n 是当前数组长度**

**同时在这里要提醒一点**。

在 JDK1.7 和 JDK1.8 的时候对 hash(key)的计算是略有不同的

**JDK1.8 时，计算 hash(key)进行了两次扰动**

**JDK1.7 时，计算 hash(key)进行了九次扰动，分别是四次位运算和五次异或运算**

**其中扰动可以理解为运算次数**

示例：

```java
h=key.hashCode()	0000 0000 0011 0110 0100 0100 1001 0010
h>>>16		        0000 0000 0000 0000 0000 0000 0011 0110
	                ---------------------------------------
hash = h^(h>>>16)	0000 0000 0011 0110 0100 0100 1010 0100
(n-1)       		0000 0000 0000 0000 0000 0000 0000 1111
(n-1) & hash			                           0100 = 4
```

![image-2](/assets/blog-image/blog2/hash2.png)

> **这么计算的目的是什么呢？**

这样设计的主要目的是为了减少**哈希冲突**，为了使计算的下标值离散化。

同时*HashMap*的容量一直都是2的倍数（扩容机制），这样的目的是为了使 (n - 1) 的二进制值 1 更多。

> **通过对hash()方法进行解析，可以拓展一个问题：**
>
> **若使用自定义对象作为key，需要注意什么呢？**

一个Object对象往往会存在多个属性字段，而选择什么属性来计算hashCode值，具有一定的考验：

- 如果选择的字段太多，而HashCode()在程序执行中调用的非常频繁，势必会影响计算性能；
- 如果选择的太少，计算出来的HashCode势必很容易就会出现重复了。

对于HashMap而言，需要重写对象key的hashCode()和equals()方法，**注意，两个都要重写**。

根据put()、hash(key)方法的研究，hashCode()值用于索引到哪个桶，equals()用于精确寻找到是哪一个Node，所以若以自定义类作为key，需要同时重写hashCode()和equals()方法。

对于JDK提供的数据分装类，例如`Integer`、`String`等，其都重写了hashCode()和equals()

- 最好不要使用自定义类作为HashMap的Key
- 如果必须要使用，一定要要重写hashCode()和equals()方法
- 重写的equals和hashCode方法中一定不能有频繁易变更的字段
- 内存缓存使用的Map，最好对Map的数据记录条数做一个强制约束，提供下数据淘汰策略。

## resize()

resize()在桶为空或桶的size超过了扩容阈值时，会被调用，对桶进行扩容，扩容后，会将原有的元素重新进行排列（因为n-1变化了）。

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 将数组扩大一倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 阈值扩大一倍
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0)//使用new HashMap(int initialCapacity)初始化后，第一次 put 的时候
            newCap = oldThr;
        else {// 对应使用 new HashMap() 初始化后，第一次 put 的时候
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // 若原数组有数据，说明不是首次初始化数组
        // 扩容后，元素需要重新分布
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 取出元素并赋值给e
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 无链则直接计算新数组中的位置
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 若为树节点则使用TreeNode的方法
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order；有链情况
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 遍历链表
                        do {
                            next = e.next;
                            // 将hash值与原size进行与运算
                            // 若为0，则不用变换位置
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else { // 若不为0，则移动到新位置
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                            //这里会将链上的元素分为结果为0和不为0两部分
                        } while ((e = next) != null);
                        // 为0的链表存储在原位置
                        // 不为0的链表存储在（原位置+原数组长度）上
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

举例说明有链的情况：

现在在数组下标为10的桶中，存有三个元素：

 ![image-3](/assets/blog-image/blog2/hash3.png)

对于这一串链表，在扩容一倍后，位置将会变化：

![image-4](/assets/blog-image/blog2/hash4.png)

## get()过程分析

get()需要根据传入的key进行获取，流程大致如下：

- 计算 key 的 hash 值，根据 hash 值找到对应数组下标: hash & (length-1)
- 判断数组该位置处的元素是否刚好就是我们要找的，如果不是，走第三步
- 判断该元素类型是否是 TreeNode，如果是，用红黑树的方法取数据，如果不是，走第四步
- 遍历链表，直到找到相等(==或equals)的 key

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 寻找到数组下标
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 判断第一个节点是不是需要的
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode) // 判断是否为树节点
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 遍历链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

## remove()过程分析

remove有两个方法：

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    
    @Override
    public boolean remove(Object key, Object value) {
        return removeNode(hash(key), key, value, true, true) != null;
    }
```

常用的方法为remove(Object key)，两个移除方法均调用的是`removeNode()`

```java
/**
    matchValue - 如果为true，则仅在值相等时删除；如果是false，则值不管相不相等，只要key和hash值一致就移除该节点。
    movable - 如果为false，则在删除时不移动其他节点
*/
	final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            // 被移除的节点、key、value
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {// 若第一个节点不是，则向下遍历链表
                if (p instanceof TreeNode)
                    // 若为树节点，则使用TreeNode方法
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    // 遍历链表
                    do {
                        // 查找目标节点
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // 若不需要判断值 || value== || value equals
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p) // p为数组桶中的节点
                    // 将其指向next
                    tab[index] = node.next;
                else
                    // 跳过p节点
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```



**参考文章：**
[深度解析 HashMap 底层实现架构 - 掘金 (juejin.cn)](https://juejin.cn/post/6985369345207566373#heading-5)

[[HashMap源码学习之路\]---数组扩容后元素的前后变化_hashmap扩容后的位置-CSDN博客](https://blog.csdn.net/wohaqiyi/article/details/81448176)

[java8-HashMap源码分析_java8 hashmap源码、-CSDN博客](https://blog.csdn.net/Yin_Tang/article/details/78571118?spm=1001.2014.3001.5501)
