---
title: HasMap底层分析
date: 2024-9-21 11:00:00 +0800
categories: [Java基础]
tags: [java]
---

# HasMap底层分析

### 一、散列表结构

HashMap的存储结构为**数组+链表+红黑树**

**同时它的数组的默认初始容量是 16、扩容因子为 0.75，每次采用 2 倍的扩容**。也就是说，每当我们数组中的存储容量达到 75%的时候，就需要对数组容量进行 2 倍的扩容。

![image-20240324103432289](/assets/blog-image/blog2/hash1.png)

初始容量和负载因子也可以通过构造方法指定：

```java
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
```



### 二、HashMap 的 put 过程

`HaahMap` 使用 `put`的方式进行数据的存储，其中有两个参数，分别是 `key `和 `value`

`HashMap` 将将要存储的值按照 `key `计算其对应的数组下标，如果对应的数组下标的位置上是没有元素的，那么就将存储的元素存放上去，但是如果该位置上已经存在元素了，那么这就需要用到我们上面所说的链表存储了，将数据按照链表的存储顺序依次向下存储就可以了。存储结果如下：

![image-20240324102513442](/assets/blog-image/blog2/hash2.png)

**链表插入：在 JDK1.7 以及前是在头结点插入的，在 JDK1.8 之后是在尾节点插入的。**

当需要存储的数据很多时，为了避免链表太长，会进行**树化**。

**树化**：当链表长度大于`8`时，会对链表进行**树化**操作，将其转换为一课**红黑树**（二叉树，左边节点的值小于根节点，右边节点的值大于根节点），这样在查找元素时类似于二分查找，提升查询效率。

当进行删除操作而使的树上的节点减少后，链表的长度不再大于`8`了，这时就要进行**链化**。而链化的条件有所不同，**只有当链表长度小于6的时候，才会将红黑树重新转化为链表**。

示意图：

![image-20240324105314892](/assets/blog-image/blog2/hash3.png)

**那为什么选取8、6这样的阈值呢？**

应用官方的说明：

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

而链化选取`6`为链化阈值，是为了防止链表长度在7、8之间来回，从而导致不断链化、树化，浪费资源。



以上为`put()`方法的整个核心过程，下面上源码：

```java
public V put(K key, V value) {
				// 对key的hashCode()做hash
        return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
      Node<K,V>[] tab; Node<K,V> p; 
      int n, i;
      //table为空，先初始化
      if ((tab = table) == null || (n = tab.length) == 0)
          n = (tab = resize()).length;
      //计算hash对应的位置，null特殊处理
      if ((p = tab[i = (n - 1) & hash]) == null)
          tab[i] = newNode(hash, key, value, null);
      else {
          Node<K,V> e; K k;
          //如果节点已经存在就替换old value
          if (p.hash == hash &&
              ((k = p.key) == key || (key != null && key.equals(k))))
              e = p;
          //bucket中是红黑树
          else if (p instanceof TreeNode)
              e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
	      //bucket中是链表
          else {
              for (int binCount = 0; ; ++binCount) {
                  if ((e = p.next) == null) {
                      p.next = newNode(hash, key, value, null);
                      //链表长度达到设定的转换为红黑树的阈值
                      if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
		                  //注意：这个方法并不是直接将链表转换为红黑树，如果当前buckets长度小于64，则扩容，否则将链表转换为红黑树
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
      //当前容量超出 loadfactor*current capacity，则扩容
      if (++size > threshold)
          resize();
      afterNodeInsertion(evict);
      return null;
  }
```

### 三、HashMap的扩容

`HashMap` 中扩容方法为`resize()`。具体源码不作分析，主要阐述扩容的理念。

`HashMap`针对于数组长度达到`负载因子*数组长度`的时候，会对数组进行`2`倍扩容。扩容后，会对数据进行重新分布，这一步是消耗性能的地方。

**扩容后数组元素的位置：**

- **无链**

  ```java
  if (e.next == null)
     newTab[e.hash & (newCap - 1)] = e;
  ```

  当数组下标有元素但未成链，则该元素按照`newTab[e.hash & (newCap - 1)]`的位置来存储。
  **扩容前：**

  ![image-20240324111448464](/assets/blog-image/blog2/hash4.png)
  **扩容后：**
  ![image-20240324111635693](/assets/blog-image/blog2/hash5.png)
  **总结：`HashMap` 扩容后，原来的元素，要么在原位置，要么在`原位置+原数组长度` 那个位置上。**

- **有链**
  盗图：

  ![image-20240324111750921](/assets/blog-image/blog2/hash6.png)
  扩容后的元素位置根上面的总结一致。

### 四、Hash(key)方法

源码：

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

先根据 `key` 的值计算到一个 `hashCode`，将 `hashCode` 的高 `16` 位二进制和低 `16` 位二进制进行**异或运算**，得到的结果再与当前数组长度减一进行与运算。最终得到一个数组下标，过程如下：

**int hashCode = key.hashCode()**

**int hash = hash(key) = key.hashCode()的高 16 位^低 16 位 &(n-1) 其中 n 是当前数组长度**

**同时在这里要提醒一点**。

在 JDK1.7 和 JDK1.8 的时候对 hash(key)的计算是略有不同的

**JDK1.8 时，计算 hash(key)进行了两次扰动**

**JDK1.7 时，计算 hash(key)进行了九次扰动，分别是四次位运算和五次异或运算**

**其中扰动可能理解为运算次数**

示例：

```java
map.put("test","1");

h=key.hashCode()	0000 0000 0011 0110 0100 0100 1001 0010
h>>>16		        0000 0000 0000 0000 0000 0000 0011 0110
	                ---------------------------------------
hash = h^(h>>>16)	0000 0000 0011 0110 0100 0100 1010 0100
(n-1)       		0000 0000 0000 0000 0000 0000 0000 1111
(n-1)& hash			                           0100 = 4
```



这里就可以解释一个`HashMap`中的现象---`HashMap`的容量一直都是`2`的倍数。

**why？**

其实这里的主要目的是为了减少`Hash冲突`(key计算的数组下标相同)。

当参与`hash(key)`算法的 `(n-1)` 的值尽可能都是`1`的时候，得到的值才是离散的。



**参考文章：**
[深度解析 HashMap 底层实现架构 - 掘金 (juejin.cn)](https://juejin.cn/post/6985369345207566373#heading-5)

[[HashMap源码学习之路\]---数组扩容后元素的前后变化_hashmap扩容后的位置-CSDN博客](https://blog.csdn.net/wohaqiyi/article/details/81448176)

[java8-HashMap源码分析_java8 hashmap源码、-CSDN博客](https://blog.csdn.net/Yin_Tang/article/details/78571118?spm=1001.2014.3001.5501)