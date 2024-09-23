---
title: LinkedList底层解析
date: 2024-9-22 11:00:00 +0800
categories: [Java基础]
tags: [java]
---

### 概述

*LinkedList*同时实现了*List*接口和*Deque*接口，也就是说它既可以看作一个顺序集合，又可以看作一个队列（*Queue*），同时还可以看作一个栈（*Stack*）。与*ArrayList*不同，*LinkedList*基于链表实现，添加和删除元素效率高。

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{}
```

### 底层数据结构

*LinkedList*的底层通过==双向连表实现==，每个节点用内部类*Node*表示。*LinkedList*通 `first` 和 `last` 引用分别指向连表的第一个和最后一个元素，当链表为空的时候，`first` 和 `last` 都指向==null==。

```java
    transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```

其中*Node*是私有的内部类

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

### 构造函数：

```java
    /**
     * Constructs an empty list.
     */    
	public LinkedList() {
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param  c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

一个无参构造和一个传入集合的构造器。

### add()

*add()*方法有两种：

- add(E e)：在LinkedList的末尾插入元素（last指向链表末尾）
- add(int index , E element)：在指定下标处插入元素

add(E e)就类似于双向链表的尾插法

```java
    /**
     * Appends the specified element to the end of this list.
     *
     * <p>This method is equivalent to {@link #addLast}.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

![linkedlist1](/assets/blog-image/blog3/linkedlist1.png)

同理，add(int index, E element)就类似于双向链表的插入，当index==size时，就相当于add(E e)；若不是，则：

1. 根据index找到要插入的位置，即node(int index)方法
2. 完成插入操作

```java
    /**
     * Inserts the specified element at the specified position in this list.
     * Shifts the element currently at that position (if any) and any
     * subsequent elements to the right (adds one to their indices).
     *
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

	// Inserts element e before non-null Node succ.
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

可以看到对于node方法，需要根据`index < size >> 1`来确定是从前往后遍历，还是从后往前遍历（靠近哪个端）。这里相较于arrayList的元素检索（数组下标检索），效率就低了。

### addAll()

```java
    /**
     * Appends all of the elements in the specified collection to the end of
     * this list, in the order that they are returned by the specified
     * collection's iterator.  The behavior of this operation is undefined if
     * the specified collection is modified while the operation is in
     * progress.  (Note that this will occur if the specified collection is
     * this list, and it's nonempty.)
     *
     * @param c collection containing elements to be added to this list
     * @return {@code true} if this list changed as a result of the call
     * @throws NullPointerException if the specified collection is null
     */
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    /**
     * Inserts all of the elements in the specified collection into this
     * list, starting at the specified position.  Shifts the element
     * currently at that position (if any) and any subsequent elements to
     * the right (increases their indices).  The new elements will appear
     * in the list in the order that they are returned by the
     * specified collection's iterator.
     *
     * @param index index at which to insert the first element
     *              from the specified collection
     * @param c collection containing elements to be added to this list
     * @return {@code true} if this list changed as a result of the call
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * @throws NullPointerException if the specified collection is null
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```

addAll()方法并未直接调用add(int indec, E element)，主要考虑到效率问题。

### get元素

*LinkedList*共有3个获取元素的方法

- getFirst()
- getLast()
- get(int index)

```java
    /**
     * Returns the first element in this list.
     *
     * @return the first element in this list
     * @throws NoSuchElementException if this list is empty
     */
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    /**
     * Returns the last element in this list.
     *
     * @return the last element in this list
     * @throws NoSuchElementException if this list is empty
     */
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
```

### remove元素

*LinkedList*共有5个删除元素的方法

- removeFirst()
- removeLast()
- remove(E e)
- remove(int index)
- void clear()

```java
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }

    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

	public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }
```

其核心都是调用 *unlink(Node<E> l)*方法：

```java
    E unlink(Node<E> x) {
        // assert x != null;
        // 待删除节点
        final E element = x.item;
        // 此节点的前后节点
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }
		// 将当前节点元素置为null，方便GC回收
        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

核心流程如下：

1. 首先获取待删除节点及其前后节点；
2. 判断待删除节点是否为头/尾节点：
   - 若 x 为头节点，则将first指向 x.next 节点
   - 若 x 为尾节点，则将last指向 x.prev 节点
   - 其他情况则执行下一步操作
3. 将 x 前一个节点的 next 指向 x 的后一个节点，将 x 的后一个节点的 prev 指向 x 的前一个节点
4. 将待 x 元素置空，方便GC回收，修改链表长度

对于 *clear()*方法，其主要是遍历链表，并将node之间的引用关系置空，方便GC回收

```java
    /**
     * Removes all of the elements from this list.
     * The list will be empty after this call returns.
     */
    public void clear() {
        // Clearing all of the links between nodes is "unnecessary", but:
        // - helps a generational GC if the discarded nodes inhabit
        //   more than one generation
        // - is sure to free memory even if there is a reachable Iterator
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
```

### 查找元素

查找的本质就是去寻找元素的下标位置，主要是采用 Object.equals来比对对象。

- indexOf(Object o)：查找第一次出现的index
- LastIndexOf(Object o)：查找最后一次出现的index

这里就是我对于LinkedList的主要分析，当然其中还实现了 *Queue* 、*Deque* 的方法，有兴趣可以去看一看，这里不做过多介绍。



参考文章：

- https://pdai.tech/md/java/collection/java-collection-LinkedList.html
- https://javaguide.cn/java/collection/linkedlist-source-code.html

