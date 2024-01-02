---
title: ArrayList详解
date: 2023-11-23 09:14:29
tags: [Java]
---
- [成员变量](#成员变量)
- [构造方法](#构造方法)
  - [无参构造](#无参构造)
  - [有参构造](#有参构造)
- [添加元素](#添加元素)
  - [扩容](#扩容)
  - [add(E e)](#adde-e)
  - [add(index, element)](#addindex-element)
  - [addAll(collection)](#addallcollection)
  - [addAll(index, collection)](#addallindex-collection)
- [删除](#删除)
  - [E remove(index)](#e-removeindex)
  - [remove(Object)](#removeobject)

## 成员变量

```java
private static final long serialVersionUID = 8683452581122892189L;

private static final int DEFAULT_CAPACITY = 10;

private static final Object[] EMPTY_ELEMENTDATA = {};

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

transient Object[] elementData; // non-private to simplify nested class access

private int size;
```

主要元素只有 size 和 elementData

## 构造方法

### 无参构造

```java
public ArrayList() {
    // private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // DEFAULTCAPACITY_EMPTY_ELEMENTDATA 是一个空 Object 数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

无参构造，大小为 0，元素指向一个空的 Object 数组

### 有参构造

_指定初始容量_

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // private static final Object[] EMPTY_ELEMENTDATA = {};
        // EMPTY_ELEMENTDATA 是一个空 Object 数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

-   initialCapacity > 0，则创建一个大小为 initialCapacity 的 Object 数组
-   initialCapacity == 0，则让数据指向一个空的 Object 数组
-   添加一个元素后，最小大小为 10，calculateCapacity 函数中 Math.max(DEFAULT_CAPACITY, minCapacity)，DEFAULT_CAPACITY = 10

_指定集合_

```java
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
```

-   集合大小为 0，则让数据指向一个空的 Object 数组
-   集合大小>0，如果集合类型是 ArrayList，那么直接指向传入的集合，如果不是则创建一个大小相同的 Object 数组并赋值

## 添加元素

### 扩容

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 新容量为就容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果新容量 < 最小需要，那么新容量为最小需要
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果新容量大于 MAX_ARRAY_SIZE(Integer.MAX_VALUE - 8)
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        // hugeCapacity 检查 minCapacity
        // 如果 minCapacity < 0，抛出 OutOfMemoryError
        // 如果 minCapacity > MAX_ARRAY_SIZE，则 minCapacity 为 Integer.MAX_VALUE，否则为 MAX_ARRAY_SIZE
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    // 创建新数组并进行赋值
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### add(E e)

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

```

```java
// minCapacity 为 size + 1
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

```java
// elementData 为此时集合中的数据
// minCapacity 为 size + 1
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // private static final int DEFAULT_CAPACITY = 10;
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

-   如果创建时使用的是无参构造，且第一次调用 add 方法，返回 max(10, size + 1)
-   否则返回 size + 1

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

-   如果当前大小小于最小要求，那么进行扩容

**总体流程**

{% asset_img add(E).png add(E)流程图 %}

### add(index, element)

```java
public void add(int index, E element) {
    // 判断 index 是否合法
    rangeCheckForAdd(index);

    // 确保大小
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 移动后面的元素
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 赋值
    elementData[index] = element;
    size++;
}
```

{% asset_img add(index,ele).png add(int,E)流程图 %}

### addAll(collection)

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

{% asset_img addAll(c).png addAll(c)流程图 %}

### addAll(index, collection)

```java
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

与上面的类似，只是加了索引范围判断，移动数据的不同

## 删除

### E remove(index)

```java
public E remove(int index) {
    // 索引范围判断
    rangeCheck(index);

    modCount++;
    // 获取该索引元素
    E oldValue = elementData(index);

    // 获取要移动的元素个数
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 将最后一个置为 null
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

### remove(Object)

```java
public boolean remove(Object o) {
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
```

流程为 for 遍历查找，然后删除

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

流程为移动元素，将最后一个置为 null
