---
layout:     post   				    
title:      数据结构回顾(四)ConcurrentHashMap1.7
subtitle:   数据结构回顾系列   #副标题
date:       2019-9-26		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-re-vs-ng2.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 数据结构回顾系列
---

# 数据结构回顾(四)ConcurrentHashMap1.7

```java
// todo
```

## 前言

上两篇笔记已经回顾了**HashMap**以及**红黑树**两种数据结构的实现，下一阶段想看的就是**ConcurrentHashMap**的1.7和1.8版本的不同实现

## Field

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    transient volatile int count;    //Segment中元素的数量
    transient int modCount;          //对table的大小造成影响的操作的数量(比如put或者remove操作)
    transient int threshold;        //阈值,Segment里面元素的数量超过这个值那么就会对Segment进行扩容
    final float loadFactor;         //负载因子,用于确定threshold
    transient volatile HashEntry<K,V>[] table;    //链表数组,数组中的每一个元素代表了一个链表的头部
}
```

```java
static final class HashEntry<K,V> {
    final K key;
    final int hash;
    volatile V value;
    final HashEntry<K,V> next;
}
```

```java
final Segment<K,V>[] segments; 

static final int DEFAULT_CONCURRENCY_LEVEL = 16; //初始的并发等级，通过并发等级来确定Segment的大小
static final int MIN_SEGMENT_TABLE_CAPACITY = 2;// segment的最小值
static final int MAX_SEGMENTS = 1 << 16; // segment的最大值
static final int DEFAULT_CONCURRENCY_LEVEL = 16; // 默认segment数量
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 默认负载因子
static final int DEFAULT_INITIAL_CAPACITY = 16; // 默认初始化容量
```

## 构造方法

```java
public ConcurrentHashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
```

```java
 public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
     // 1 
     if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
          throw new IllegalArgumentException();
    
      if (concurrencyLevel > MAX_SEGMENTS)
          concurrencyLevel = MAX_SEGMENTS;
      // 2
      int sshift = 0;
      int ssize = 1;
      while (ssize < concurrencyLevel) {
           ++sshift;
           ssize <<= 1;
       }
       segmentShift = 32 - sshift;
       segmentMask = ssize - 1;
       this.segments = Segment.newArray(ssize);
       // 3
       if (initialCapacity > MAXIMUM_CAPACITY)
           initialCapacity = MAXIMUM_CAPACITY;
       int c = initialCapacity / ssize;
       if (c * ssize < initialCapacity)
           ++c;
       int cap = 1;
       while (cap < c)
           cap <<= 1;
       
       Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
       Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
       UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
       this.segments = ss;
 }
```

**步骤1：**对传入的初始化容量，负载因子，segment数量进行校验

**步骤2：**根据传入的**concurrencyLevel**计算ssize用于segment数组的初始化，由此步可见，segment数组的长度是不大于**concurrencyLevel**的2的倍数，并且在这一步记录下了两个成员变量**segmentShift**和**segmentMask**

* **segmentShift = **32 - log2(ssize)
* **segmentMask = ** ssize - 1 

**步骤3：**计算initialCapacity和ssize的整数比，然后cap也就是segment数组中table的长度就是不大于这个整数比的2的倍数，然后用计算好的ssize和cap两个值创建segments数组和segment[0]的table数组

## [ConcurrentHashMap-> put]

```java
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        // 1
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

**步骤1：**通过位运算计算出一个hash值后会右移32 - log2(segmentSize)之后&segmentSize - 1得到一个值j，j再通过与static块中定义好的静态变量运算，拿到对应segment



## [ConcurrentHashMap-> get]

## [ConcurrentHashMap-> remove]







参考资料：

[Java泛型底层源码解析](https://www.cnblogs.com/liang1101/p/6407871.html)