---
title: Java集合源码之ArrayList
categories:
  - Java
tags:
  - Java
  - 集合
  - ArrayList
  - 源码
cover: 'https://cdn.littlecorgi.top/blog/ArrayList%E5%B0%81%E9%9D%A2%E5%9B%BE.jpg'
abbrlink: 970dc297
date: 2019-11-18 22:14:10
updated: 2019-11-18 22:14:10
---
# 简介
ArrayList可以说是我们最常用的一种集合了。

他的本质是一个数组，一个**可以自动扩容的动态数组**，**线程不安全**，**允许元素为null**。

由于数组的内存连续，可以根据下标以O(1)的时间读写元素，因此时间效率很高。

# 内部属性
我们先来看下ArrayList里面有哪几个属性:
* `private static final long serialVersionUID = 8683452581122892189L;`
序列话UID。由于ArrayList实现了Serializable接口，为了序列化和反序列化的方便，我们就手动为他添加一个序列化UID。

* `private static final int DEFAULT_CAPACITY = 10;`
默认的容量。

* `private static final Object[] EMPTY_ELEMENTDATA = {};`
空数组。

* `private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};`
默认的空数组。

* `transient Object[] elementData;`
真正存放元素的数组。

* `private int size;`
当前元素个数。

# 构造方法
```java
// 传入参数为初始化容量时的构造方法
public ArrayList(int initialCapacity) {

    if (initialCapacity > 0) {
        // 如果传入参数大于零，那就创建一个对应大小的数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // 如果传入参数等于0，那就直接把属性中创建好的空数组复制
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        // 如果传入参数小于0，那就抛异常
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

// 传入参数为空时的构造方法
public ArrayList() {
    // 将属性中创建好的空数组复制过来
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 传入参数为数组时的构造方法
public ArrayList(Collection<? extends E> c) {
    // 先转换为数组
    elementData = c.toArray();
    // 因为size代表ArrayList中元素个数，所以要把数组的长度赋过来
    // 如果个数不为0进入此if
    if ((size = elementData.length) != 0) {
        //这里是当c.toArray出错，没有返回Object[]时，利用Arrays.copyOf 来复制集合c中的元素到elementData数组中
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 如果个数为0，那就把属性中的空数组复制过来
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

# 常用API
## 增
### ArrayList # add(E e)
```java
public boolean add(E e) {
    // ->> 分析4.1.1
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 将需要加入的元素加到最后面
    elementData[size++] = e;
    return true;
}

/**
 * 分析4.1.1 ensureCapacityInternal()
 */
private void ensureCapacityInternal(int minCapacity) {
    // DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个变量只有在通过无参构造的时候用到过
    // 也就是说，他判断创建出来的这个数组，到底是不是无参构造方法创建出来的，如果是就找出DEFAULT_CAPACITY和minCapacity中较大的那个
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 但是按道理来说，add一个元素minCapacity肯定为1，肯定小于DEFAULT_CAPACITY，为什么还要做一个判断呢？
        // 但是你得注意，还有AddAll这个方法
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    
    // ->> 分析4.1.2
    ensureExplicitCapacity(minCapacity);
}

/**
 * 分析4.1.2 ensureExplicitCapacity()
 */
private void ensureExplicitCapacity(int minCapacity) {
    // modCount是AbstractList中的属性，如果需要扩容，则会修改modCount
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        // ->> 分析4.1.3
        grow(minCapacity);
}

/**
 * 分析4.1.3 grow()
 * 作用：扩容
 */
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 扩容为原数组的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果还不够，就直接用能容纳的最小大小
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8
    // 如果新数组比MAX_ARRAY_SIZE还要大的话
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        // ->> 分析4.1.4
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually clos2019e to size, so this is a win:
    // 生成新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}

/**
 * 分析4.1.4
 */
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

### ArrayList # add(E e)
```java
// 按给定的位置添加指定元素
public void add(int index, E element) {
    // 如果给的位置超过了集合已存放的元素的个数或者小于0，就抛异常
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    // ->> 分析4.1.1
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 将elementData从index处分开，给index空出位置
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 放入数据
    elementData[index] = element;
    size++;
}
```

### ArrayList # addAll(Collection<? extends E> c)
```java
public boolean addAll(Collection<? extends E> c) {
    // 转为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    // 扩容数组 ->> 分析4.1.1
    ensureCapacityInternal(size + numNew);  // Increments modCount
    // 将a数组的全部内容添加到elementData数组从size开始之后
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

### ArrayList # addAll(int index, Collection<? extends E> c)
```java
public boolean addAll(int index, Collection<? extends E> c) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    // 转为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    // 扩容 ->> 分析4.1.1
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    // 如果index小于size，也就是说需要在原数组的中间插入的haul，就需要先把原数组index之后的数据往后移动
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

### 总结
* 无论是add还是addAll，都是先判断是否越界，如果越界就扩容，然后再移动数组
* 如果需要扩容，默认扩容原来的一般大小；如果还不够，那就直接将目标的size作为扩容后的大小
* 在扩容成功后，会修改modCount

## 删
### ArrayList # remove(int index)
```java
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    // 修改modCount
    modCount++;
    // 将要删除的内容先保存下来
    E oldValue = (E) elementData[index];

    int numMoved = size - index - 1;
    // 判断要删除的数据是不是最后一位
    // 如果不是最后一位，还得先把后面的数据往前移动
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 清除数据，更改引用，让GC去清理
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

### ArrayList # remove(Object o)
```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                // ->> 分析4.2.1
                fastRemove(index);
                return true;
            }
    } else {
        // 如果参数不为空，那就遍历整个数组集合
        for (int index = 0; index < size; index++)
            // 找到和参数相等的那一位，然后将该位移除
            if (o.equals(elementData[index])) {
                // ->> 分析4.2.1
                fastRemove(index);
                return true;
            }
    }
    return false;
}

/** 
 * 分析4.2.1：fastRemove()
 * 作用：基本上和remove(int index)一样
 */
private void fastRemove(int index) {
    // 修改modCount
    modCount++;
    int numMoved = size - index - 1;
    // 判断要删除的数据是不是最后一位
    // 如果不是最后一位，还得先把后面的数据往前移动
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 清除数据，更改引用，让GC去清理
    elementData[--size] = null; // clear to let GC do its work
}
```

### ArrayList # removeAll(Collection<?> c)
```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    // ->> 分析4.2.2
    return batchRemove(c, false);
}

/** 
 * 分析4.2.2 batchRemove()
 */
private boolean batchRemove(Collection<?> c, boolean complement) {
    // 先赋值
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        // 快速保存两个集合共有元素
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            // 如果出现异常会导致r!=size，就把异常之后的数据全部覆盖到数组里面
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
            // 将后面的元素全部置空，让GC来回收
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```
这块我看的时候有点绕，但是一想通就好了。

其实这个地方是要把C集合中和原集合中共有的元素删除，那我就只需要遍历原数组，然后碰到和C集合相同的元素就直接放到前面去，然后等遍历完成后，一次性把后面的全部置空。

### ArrayList # retainAll(Collection<?> c)
```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    // ->> 分析4.2.2
    return batchRemove(c, true);
}
```

### ArrayList # clear()
```java
public void clear() {
    modCount++;

    // clear to let GC do its work
    // 直接遍历每一位，然后把每一位都置空，让GC去清理
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

### 总结
* 所有的删除操作都会修改modCount

## 改
```java
public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    // 取出原来的元素
    E oldValue = (E) elementData[index];
    // 把需要更改的数据放进去
    elementData[index] = element;
    return oldValue;
}
```
没啥好分析的

不需要修改modCount，相对高效

## 查
```java
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index];
}
```
没啥好分析的

不需要修改modCount，相对高效

## 包括 contains() & indexOf()
```java
public boolean contains(Object o) {
    // ->> 分析4.5.1
    return indexOf(o) >= 0;
}

/**
 * 分析4.5.1：indexOf()
 */
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
```
没啥好分析的

## 判空 isEmpty()
```java
public boolean isEmpty() {
    return size == 0;
}
```
没啥好分析的

# 迭代器 Iterator
## 创建迭代器
```java
public Iterator<E> iterator() {
    // 构造Itr对象并返回
    return new Itr();
}
```

## Itr属性
```java
// 限制，也就是数组的元素个数
protected int limit = ArrayList.this.size;

// 下一个元素的下标
int cursor;       // index of next element to return
// 上一次返回元素的下标
int lastRet = -1; // index of last element returned; -1 if no such
// 用于判断集合是否修改过结构的标志
int expectedModCount = modCount;
```

## Itr # hasNext()
```java
public boolean hasNext() {
    return cursor < limit;
}
```
不用多说

## Itr # next()
```java
public E next() {
    // 判断是否修改过List的结构，如果修改了就抛异常
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    int i = cursor;
    // 如果越界了就抛异常
    if (i >= limit)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    // 再次判断是否越界，在 我们这里的操作时，有异步线程修改了List
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    // 标记加1
    cursor = i + 1;
    // 返回数据，并设置上一次的下标
    return (E) elementData[lastRet = i];
}
```

## Itr # remove()
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();

    try {
        // 调用ArrayList的remove方法移除数据
        ArrayList.this.remove(lastRet);
        // 更新一系列数据
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
        limit--;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

# 总结
## modCount到底是个什么东西？
如果你看过很多集合源码的话，你就会发现你会在很多地方都会碰到这个modCount，例如ArrayList、LinkedList、HashMap等等。modCount字面意思就是修改次数，那么为什么需要纪录这个修改次数呢？

你回想下，似乎用到modCount的地方，如ArrayList、LinkedList等等，他们都用一个共性——线程不安全。

而在你看了这几个类的迭代器后就会发现，他们迭代器中一定有一个属性(如我们上面的expectedModCount)初始化时的值就是modCount的值。

然后在迭代器的方法中，只要是遍历这个集合的时候，都需要将两个值进行对比，然后再移除元素的时候，都需要更新迭代器里面的modCount变量。

这时，你需要了解下**Fail-Fast机制**。

我们都知道这些集合是线程不安全的，如果在使用迭代器的过程中，有其他线程对集合进行了修改，那么就会抛出ConcurrentModificationException异常，这就是Fail-Fast策略。而这个时候源码中就通过modCount进行了操作。迭代器在创建时，会创建一个变量等于当时的modCount，如果在迭代过程中，集合发生了变化，modCount就是++。这时迭代器中的变量的值和modCount不相等了，那就抛异常。

所以，**遍历线程不安全的集合时，尽量使用迭代器**

解决办法：
* 在遍历过程中所有涉及到改变 modCount 值得地方全部加上 synchronized 或者直接使用 Collections.synchronizedList，这样就可以解决。但是不推荐，因为增删造成的同步锁可能会阻塞遍历操作。
* 使用 CopyOnWriteArrayList 来替换 ArrayList。推荐使用该方案。关于CopyOnWriteArrayList的内容此处不再过多的去将，想了解的同学可以百度或者谷歌。

## ArrayList和Vector的区别
* ArrayList线程不安全，Vector线程安全
* 扩容的时候ArrayList默认扩容1.5倍，Vector默认扩容1倍

