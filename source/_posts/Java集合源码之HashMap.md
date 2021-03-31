---
title: Java集合源码之HashMap
categories:
  - Java
tags:
  - Java
  - 集合
  - HashMap
  - 源码
cover: 'https://cdn.littlecorgi.top/blog/HashMap%E5%B0%81%E9%9D%A2%E5%9B%BE.jpg'
abbrlink: 2982320f
date: 2019-11-22 20:11:15
updated: 2019-11-22 20:11:15
---
# 简介
HashMap是一个哈希表，线程不安全，`key`唯一，`value`可重复，允许`key`和`value`为null。遍历时是无序的。

底层结构是基于链表散列，也就是数组+链表。数组也被称为哈希桶，桶里面就装着链表，链表中的每个节点，就是哈希表中的每个元素。

在JDK8中，当链表长度达到8的时候，就会转为红黑树。

它实现了`Map<K, V>, Cloneable, Serializable`接口。

接下来我们就来看下源码：

# 属性
```java
// 序列化ID，用于序列化和反序列化
private static final long serialVersionUID = 362498820763181265L;

// 默认初始容量也就是16-必须为2的幂。
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大容量。
// 如果两个构造函数都使用参数隐式指定了更高的值，则使用该容量。 
// 必须是2的30次方。
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认的负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// Entry数组，也就是哈希桶，长度为2的n次幂
transient Node<K,V>[] table;

/**
 * 保存缓存的entrySet()。 注意，AbstractMap字段用于keySet（）和values（）。
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * 包含的键-值对的数，也就是元素的个数
 */
transient int size;

// 这个与fast-fail有关，可以参考我上一篇将ArrayList部分博客的modCount变量。
transient int modCount;

// 当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容
int threshold;

// 负载因子
final float loadFactor;
```

这些可能现在看起来还很迷，可能都不知道有些变量到底是拿来干嘛的。不急，我们继续往后面看。

# 构造方法
```java
// 构造一个指定初始容量和构造因子的HashMap
public HashMap(int initialCapacity, float loadFactor) {
    // 如果指定的初始容量小于0，抛异常
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // 如果指定的初始容量大于设定好的最大容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        // 就直接把初始容量设置为最大容量
        initialCapacity = MAXIMUM_CAPACITY;    // 如果负载因子小于0或者为空，抛异常
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    // 赋值
    this.loadFactor = loadFactor;
    // ->> 分析3.1
    this.threshold = tableSizeFor(initialCapacity);
}

// 构造一个具有指定容量和默认负载因子的HashMap。
public HashMap(int initialCapacity) {
    // 调用了上面那个构造方法
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 无参构造方法
public HashMap() {
    // 直接加载默认负载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

// 将Map里面的值传入HashMap
public HashMap(Map<? extends K, ? extends V> m) {
    //  使用默认的加载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 分析3.2
    putMapEntries(m, false);
}
```

## 分析3.1 HashMap # tableSizeFor(int cap)
```java
/**
 * 分析 3.1
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    // >>> 无符号右移
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
这个方法有点纠结，你光看代码确实看不出来个啥，但是如果你在纸上顺便找几个例子写写就懂了。

他的用途，总结起来就是：**找到大于或等于cap的最小2的幂**。

这里借用一下[一位大佬的图](http://www.tianxiaobo.com/2018/01/18/HashMap-%E6%BA%90%E7%A0%81%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90-JDK1-8/)：
![](https://image-static.segmentfault.com/327/110/3271107614-5a65971be833a)

这个图就是第一个例子。cap=536870913，经过运算之后，n+1=1073741824。

## 分析3.2 HashMap # putMapEntries(Map<? extends K, ? extends V> m, boolean evict)
```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    // 先拿到map的大小
    int s = m.size();
    // 如果map大于0才去进行操作，如果等于0那就证明Map是空的，就不进行操作
    if (s > 0) {
        // 如果当前哈希表还是空的
        if (table == null) { // pre-size
            // 根据m的元素数量和当前表的加载因子，计算出阈值
            float ft = ((float)s / loadFactor) + 1.0F;
            // 修正阈值的边界 不能超过MAXIMUM_CAPACITY
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            // 如果新的阈值大于当前阈值
            if (t > threshold)
                // 返回一个新的阈值的 满足2的n次方的阈值
                // ->> 分析3.1
                threshold = tableSizeFor(t);
        }
        // 如果当前哈希表不是空的，而且map中元素个数大于当前的阈值，那就重新扩容
        else if (s > threshold)
            // 扩容
            // ->> 分析3.3
            resize();
        // 遍历Map，取出map每一个键值对并复制给HashMap
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            // 将取出Key和Value放入HashMap
            // ->> 分析3.4
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

## 分析3.3 HashMap # resize()
**重点！！！**

扩容函数。

先来看源码
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 旧数组也就是旧哈希桶的容量，如果oldTab为空就为0，否则为oldTab长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧的阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 如果旧哈希桶容量大于0
    if (oldCap > 0) {
        // 如果旧哈希桶容量大于限定的最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 阈值就设置为int的最大值
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 否则新的容量是旧的容量的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                // 旧的容量为默认初始化容量也就是16
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新的阈值也等于旧的阈值
            newThr = oldThr << 1; // double threshold
    }
    // 如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 直接把新的容量的值设置为旧的阈值
        newCap = oldThr;
    // 如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况
    else {               // zero initial threshold signifies using defaults
        // 新容量为默认的初始容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        // 新的阈值默认为默认负载因子 * 默认初始容量 = 16 * 0.7 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果新的阈值是0，对应的是  当前表是空的，但是有阈值的情况
    if (newThr == 0) {
        // 根据新表容量 和 加载因子 求出新的阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 更新阈值 
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 创建新的哈希桶
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 更新引用
    table = newTab;
    // 如果以前的哈希桶中还有元素，那就把原来的数组复制过来
    if (oldTab != null) {
        // 遍历以前的哈希桶
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 如果当前节点不为空
            if ((e = oldTab[j]) != null) {
                // 把旧哈希桶中的该位置为空，方便GC回收
                oldTab[j] = null;
                // 如果该节点的后一位为空了，也就是说哈希桶里面的链表只有一个节点
                if (e.next == null)
                    //直接将这个元素放置在新的哈希桶里。
                    //注意这里取下标 是用 哈希值 与 桶的长度-1 。 由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高
                    // 具体会在文章最后面讲
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树（暂且不谈 避免过于复杂， 后续专门研究一下红黑树）
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。
                else { // preserve order
                    // 由于现在容量加倍了，按照哈希表的算法，之前在这个哈希桶里面的现在不一定属于该哈希桶了
                    // 比如，之前哈希长度是4，有一个元素为5
                    // 未扩容前，他应该在5%4=1这个桶里面
                    // 但是现在扩容了，哈希长度变成了8，那么他现在就应该在5%8=5这个桶里面了
                    // 而且新桶的位置就是老位置1加上oldCap4，也就是j + oldCap = 1 + 4 = 5
                    
                    // 代表扩容后还应该在老桶里面的元素
                    Node<K,V> loHead = null, loTail = null;
                    // 代表扩容后应该在新桶里面的元素
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 这里又是一个利用位运算 代替常规运算的高效点： 利用哈希值 与 旧的容量，可以得到哈希值去模后，是大于等于oldCap还是小于oldCap，等于0代表小于oldCap，应该存放在低位，否则存放在高位                    
                        if ((e.hash & oldCap) == 0) {
                            // 以下均为简单的单链表操作
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            // 以下均为简单的单链表操作
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 遍历两个链表，把lo放到原来的位置上，并把hi放入新位置也就是j+oldCap
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

总结来说就是:
* 扩容
    * 如果之前的哈希表有值，而且元素的个数大于限定的哈希表最大容量的话，那就把新的哈希表的阈值设置为Integer.MAX_VALUE也就是int的最大值，并直接返回旧和哈希表
    * 如果之前的哈希表有值，而且元素个数没有大于哈希表最大容量，而且个数的两倍还小于限定的哈希表最大容量、元素个数大于默认的初始容量也就是16的话，就设置新的阈值为旧的阈值的2倍
    * 如果之前的哈希表的空的，但是阈值是存在值的，也就相当于你初始化时指定了容量和阈值然后构造的哈希表，此时就直接让新的容量指定为旧的阈值
    * 如果之前哈希表是空的，阈值也不大于0，也就相当于采用了无参构造，这时就直接让新容量为默认的初始容量也就是16，让新的阈值为默认负载因子*默认初始容量 = 16 * 0.7 = 12
    * 如果新的阈值是0的话，也就是在之前的4个判断中没有对新的阈值进行更改的情况，也就是之前的哈希表是空的，然后指定了阈值的情况，这是就根据新表的容量和负载因子求出新的阈值
* 复制到新表
    * 如果以前的哈希表中有元素，就直接遍历原表，然后判断当前哈希桶里面有没有元素
    * 如果有而且他的下一个为空，就代表这个链表只有一个节点，那就直接求出他的哈希值，然后给对应的哈希桶
    * 如果他下一个不为空而且他是红黑树的实例，那就调用split方法
    * 如果他下一个不为空，也不是红黑树的实例，那就开始遍历链表，判断当前及节点新的哈希值，如果还是当前值，那就放到lo链表上去，如果不是当前值，那就放到hi链表
    * 然后遍历完成后，直接把lo链表放到当前哈希桶，把hi放到当前哈希值+旧的哈希数组长度也就是新的哈希值对应的哈希桶上去

## 分析3.4 HashMap # putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果哈希表是空的的话，代表是初始化的
    if ((tab = table) == null || (n = tab.length) == 0)
        // 那就直接扩容哈希表，并且将扩容后的哈希桶长度赋值给n
        n = (tab = resize()).length;
    // 如果当前index的节点是空的，表示没有发生哈希碰撞。 直接构建一个新节点Node，挂载在index处即可。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 发生哈希碰撞
        Node<K,V> e; K k;
        // 如果哈希值相等、key也相等，那就直接覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果是红黑树的实例，暂时不讨论
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 如果以上情况都不满足，也就代表不是红黑树，但是该链表不止一个节点
        else {
            // 对链表进行遍历
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 放入链表最后面
                    p.next = newNode(hash, key, value, null);
                    // 如果当前链表节点个数大于8
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 转为红黑树
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果找到了要覆盖的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果e不为null，也就是说找到了需要覆盖的节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 如果onlyIfAbsent为false，或者oldValue不存在才进入if
            // onlyIfAvsent这个变量的意思是如果为true，则此key对应的节点有value的情况，否则是没有value的情况，此变量用于putIfAbsent方法
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 这是一个空实现的函数，用作LinkedHashMap重写使用
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 由于节点插入成功，或者覆盖成功，所以需要修改modCount通知HashMap结构发生变化
    ++modCount;
    // 更新size，如果size达到了阈值，那就扩容
    if (++size > threshold)
        resize();
    // 这是一个空实现的函数，用作LinkedHashMap重写使用
    afterNodeInsertion(evict);
    return null;
}
```

# 常用API
## 增、改
### HashMap # put(K key, V value)
功能：
* 如果原来的表里面没有key对应的节点，就把key和value插入
* 如果有对应的节点，就把将key对应节点的value更改为传入的value


```java
public V put(K key, V value) {
    // ->> 见分析3.4
    return putVal(hash(key), key, value, false, true);
}
```

### HashMap # putAll(Map<? extends K, ? extends V> m)
功能：
* 直接把Map里面的内容传入HashMap


```java
public void putAll(Map<? extends K, ? extends V> m) {
    // ->> 分析3.2
    putMapEntries(m, true);
}
```

### HashMap # putIfAbsent(K key, V value)
功能：
* 根据key找到对应节点
* 如果该节点已经有value，就返回此value
* 否则就将此value插入，并返回null


```java
@Override
public V putIfAbsent(K key, V value) {
    // ->> 分析3.4
    return putVal(hash(key), key, value, true, true);
}
```

### HashMap # replace(K key, V oldValue, V newValue)
功能：
* 根据key和value找到对应节点，然后把旧值用newValue替换


```java
@Override
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    // ->> 分析4.3.1
    // 根据key找到对应的节点，并判断节点为不为空，以及此时的value的值等不等于传入的oldValue
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        // 替换为新值
        e.value = newValue;
        // 这是一个空实现的函数，用作LinkedHashMap重写使用
        afterNodeAccess(e);
        return true;
    }
    return false;
}
```

### HashMap # replace(K key, V value)
功能：
* 根据key找到对应的节点，然后直接把value替换过去

```java
@Override
public V replace(K key, V value) {
    Node<K,V> e;
    // ->> 分析4.3.1
    // 找到对应的节点，并判断为不为空
    if ((e = getNode(hash(key), key)) != null) {
        // 取出之前的值
        V oldValue = e.value;
        // 替换为新的值
        e.value = value;
        // 这是一个空实现的函数，用作LinkedHashMap重写使用
        afterNodeAccess(e);
        // 把旧值返回
        return oldValue;
    }
    return null;
}
```

## 删
### remove(Object key)
功能：
* 根据key找到节点并删除

```java
public V remove(Object key) {
    Node<K,V> e;
    // ->> 分析4.2.1
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

### remove(Object key, Object value)
功能：
* 根据key和value找到对应节点，然后删除

```java
public boolean remove(Object key, Object value) {
    // ->> 分析4.2.1
    return removeNode(hash(key), key, value, true, true) != null;
}
```

### 分析4.2.1 HashMap # removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable)
```java
/**
 * 分析4.2.1：HashMap.removeNode()
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 如果哈希桶不为空并且哈希桶的长度大于0并且key对应的哈希桶不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 如果链表第一个节点就是要删除的节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 直接赋值给node
            node = p;
        // 否则的话，看下一位是否为空，如果不为空，那就证明此处有链表或者红黑树
        else if ((e = p.next) != null) {
            // 红黑树
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // 不是红黑树，就代表只是链表，而且链表长度不超过8
            else {
                // 遍历找到对应的节点，然后赋值给node
                do {
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
        // 找到了节点并且要么matchValue为false，要么node的value得和传入的value相等
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 红黑树，不管
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果node==p也就是该哈希桶里面第一个节点就是我们需要找的
            else if (node == p)
                // 操作
                tab[index] = node.next;
            // 上面if不成立也就代表着要找的节点不是哈希桶里面的第一个节点
            else
                // 操作
                p.next = node.next;
            // 由于我们改变了哈希表的结构，所以需要更新modCount
            ++modCount;
            // 更新size
            --size;
            // 这是一个空实现的函数，用作LinkedHashMap重写使用
            afterNodeRemoval(node);
            return node;
        }
    }
    // 在对应哈希桶里面找不到对应的节点，就返回null
    return null;
}
```


## 查
### HashMap # get(Object key)
功能：
* 根据key找到对应节点，并返回节点的value。


```java
public V get(Object key) {
    Node<K,V> e;
    // ->> 分析4.4.1
    // 找得到节点就返回节点的value，找不到则返回null
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

### HashMap # containsValue(Object value)
功能：
* 根据value找到第一个找到的节点；
* 如果找到了就返回true；
* 没有返回false。


```java
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    // 如果哈希表由内容的话进入if
    if ((tab = table) != null && size > 0) {
        // 直接遍历哈希桶数组
        for (int i = 0; i < tab.length; ++i) {
            // 再遍历当前哈希桶里面的节点，判断有没有需要找的value，如果有就返回，没有就遍历下一个哈希桶
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```

### HashMap # containsKey(Object key)
功能：
* 根据key找到对应节点；
* 如果有返回true；
* 否则返回false。


```java
public boolean containsKey(Object key) {
    // ->> 分析4.3.1
    return getNode(hash(key), key) != null;
}
```

### HashMap # getOrDefault(Object key, V defaultValue)
功能：
* 根据key找到对应节点；
* 如果有返回找到节点的value；
* 否则返回传入的defaultValue。


```java
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    // ->> 分析4.3.1
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```

### 分析4.3.1 HashMap # getNode(int hash, Object key)
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 哈希表里面有数据
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 如果是该哈希桶里面的第一个节点就是要找的节点，那么直接将该节点返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 如果第一个节点不是，那就遍历后续节点
        if ((e = first.next) != null) {
            // 红黑树，不做讨论
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在节点数小于等于8时，此时还没转为红黑树，这时就遍历该链表，找到要找的节点，并返回
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

## 判空 isEmpty()
这个方法没啥好说的

```java
public boolean isEmpty() {
    return size == 0;
}
```


## 遍历
先将一下用法吧。因为HashMap的遍历有点特别。

他的遍历并不是直接使用迭代器或者他自己有个啥foreach方法能遍历(虽说他有foreach方法，但是他那个方法不是用来遍历，而是遍历进行操作的)。

他的遍历方式很奇特：
```java
for(Object key : map.keySet()) {
    // do something
}
```
或
```java
for(HashMap.Entry entry : map.entrySet()) {
    // do something
}
```

从上面代码片段中可以看出，大家一般都是对 HashMap 的 key 集合或 Entry 集合进行遍历。上面代码片段中用 foreach 遍历 keySet 方法产生的集合，在编译时会转换成用迭代器遍历，等价于：
```java
Set keys = map.keySet();
Iterator ite = keys.iterator();
while (ite.hasNext()) {
    Object key = ite.next();
    // do something
}
```

大家在遍历 HashMap 的过程中会发现，多次对 HashMap 进行遍历时，遍历结果顺序都是一致的。但这个顺序和插入的顺序一般都是不一致的。产生上述行为的原因是怎样的呢？

现在咱们就先来看看代码吧。

首先看下keySet方法：

### HashMap # keySet()
```java
public Set<K> keySet() {
    // 复制，尽管我在HashMap类里面没有找到这个变量，但是不妨碍我们看源码
    Set<K> ks = keySet;
    if (ks == null) {
        // 构造方法
        // ->> 4.5.2
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}
```

### HashMap # KeySet
```java
final class KeySet extends AbstractSet<K> {
    // 返回HashMap的size
    public final int size()                 { return size; }
    // 调用HashMap的clear方法
    public final void clear()               { HashMap.this.clear(); }
    // ->> 4.5.3
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    // 调用HashMap的containsKey方法
    public final boolean contains(Object o) { return containsKey(o); }
    // 调用HashMap的removeNode方法
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            // fast-fail 可以看我之前ArrayList源码博客
            int mc = modCount;
            // Android-changed: Detect changes to modCount early.
            // 遍历每一个哈希桶
            for (int i = 0; (i < tab.length && modCount == mc); ++i) {
                // 遍历哈希桶里面每一个节点
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            // 在遍历过程中，如果有其他线程对此HashMap操作导致HashMap的结构发生了变化，导致modCount的值发生了变化，进而不等于mc，抛异常
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

### HashMap # KeyIterator
```java
final class KeyIterator extends HashIterator
    implements Iterator<K> {
    // nextNode()是HashIterator里的方法
    public final K next() { return nextNode().key; }
}
```

### HashMap # HashIterator
```java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        // 找到第一个包含节点的桶
        if (t != null && size > 0) { // advance to first entry
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        // fast-fail
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            // 寻找下一个包含节点的桶
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        // fast-fail
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        // 调用HashMap的removeNode方法
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```

如上面的源码，遍历所有的键时，首先要获取键集合KeySet对象，然后再通过 KeySet 的迭代器KeyIterator进行遍历。KeyIterator 类继承自HashIterator类，核心逻辑也封装在 HashIterator 类中。HashIterator 的逻辑并不复杂，在初始化时，HashIterator 先从桶数组中找到包含链表节点引用的桶。然后对这个桶指向的链表进行遍历。遍历完成后，再继续寻找下一个包含链表节点引用的桶，找到继续遍历。找不到，则结束遍历。


# 拓展
## 为什么哈希桶的长度一定是2的次方幂
这是因为在哈希表中，为了计算方便，他全部都用`hash&(n-1)`代替了`hash%n`，但是这样替换的前提就是n必须是2的次方幂。

## hash()方法核心
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);    //key.hashCode()为哈希算法，返回初始哈希值
}
```
代码的流程就是执行key的hashCode方法得到他的hashCode，然后再让他的hashCode与hashCode右移16位之后的值相与。

那为啥要这样呢？

理论上散列值是一个int型，如果直接拿散列值作为下标访问HashMap主数组的话，考虑到2进制32位带符号的int表值范围从-2147483648到2147483648。前后加起来大概40亿的映射空间。只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。

但问题是一个40亿长度的数组，内存是放不下的。你想，HashMap扩容之前的数组初始大小才16。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来访问数组下标。

这样的话，就算我的散列值分布的再松散，要是只取最后几位的话，碰撞也会很严重，这时，就可以让hashCode自己的高位和低位相与，这样加大了数据的随机性，很大程度降低了碰撞的几率。
![](https://cdn.littlecorgi.top/mweb/2019-11-22/15744333648313.jpg)


## 变量table为什么被transient修饰
HashMap明明实现了serializable接口，而且还定义里serialVersionUID，那为什么table还需要用transient修饰而不去直接序列化呢？

其实HashMap并没有使用默认的序列化机制，而是通过实现readObject/writeObject两个方法自定义了序列化的内容。如果直接对table进行序列化的话存在着两个问题:
* table大多数情况下都是无法被存满的，序列化未使用的部分，浪费空间；
* 同一个键值对在不同JVM下，所处的桶位置可能是不同的，在不同的JVM下反序列化table可能会发生错误。

第一个好理解，那我们说一下第二个：大家都知道HashMap很多地方都是根据hash值来进行存入或者取出数据的，但是hash调用的是Object的hashCode方法。但是hashCode方法是native方法，这个是取决于JVM的，不同的JVM可能计算hashCode的方法不同。所以如果在不同的平台下，序列化和反序列化HashMap之后。可能序列化时的HashMap是对着的，直接反序列化后，可能序列化中的元素a应该放在哈希桶1中，这样，反序列化也放在桶1中，但是由于反序列化的平台不同，导致hashcode算法不同，导致在反序列时元素a应该在桶2中，这样就导致HashMap结构出问题。