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
        // 2
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        // 3
        return s.put(key, hash, value, false);
    }
```

**步骤1：**通过位运算计算出一个hash值后会右移32 - log2(segmentSize)之后&segmentSize - 1得到一个索引值j，

**步骤2：**索引值j再通过与static块中定义好的静态变量运算获得地址，拿到对应segment，如果segment为空则调用**ensureSegment**方法创建对应的segment[j]

**步骤3：**在对应的segment中插入数据

## [ConcurrentHashMap-> ensureSegment]

```java
private Segment<K,V> ensureSegment(int k) {
        // 1
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        // 2
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            // 3
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
 }
```

**步骤1：**采用和**ConcurrHashMap-> put**相同的方式计算地址值

**步骤2：**由步骤1中计算出的地址值可见性地获取segment[k]即segment[j]，如果为空则按照segment[0]的HashEntry数组长度和加载因子初始化segment[k]，这也解释了为什么在构造函数中只创建segment[0]，这是因为其他segment创建的模版就是segment[0]，而其他segment是延时创建的，只有在使用的时候才创建

**步骤3：**重新可见性检查segment[k]是否存在，防止在步骤2中有其他线程创建了segment[k]，然后用CAS的方式给segment[k]赋值

## [Segment-> put]

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
         // 1
         HashEntry<K,V> node = tryLock() ? null :
              scanAndLockForPut(key, hash, value);
          V oldValue;
          try {
             // 2
             HashEntry<K,V>[] tab = table;
             int index = (tab.length - 1) & hash;
             HashEntry<K,V> first = entryAt(tab, index);
             // 3 
             for (HashEntry<K,V> e = first;;) {
                 if (e != null) {
                     K k;
                     if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                          oldValue = e.value;
                          if (!onlyIfAbsent) {
                             e.value = value;
                             ++modCount;
                          }
                          break;
                     }
                     e = e.next;
                 }
                 else {
                     // 4 
                     if (node != null)
                         node.setNext(first);
                     else
                         node = new HashEntry<K,V>(hash, key, value, first);
                     // 5
                     int c = count + 1;
                     if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                         rehash(node);
                     else
                         setEntryAt(tab, index, node);
                     ++modCount;
                     count = c;
                     oldValue = null;
                     break;
                }
            }
       } finally {
             unlock();
       }
       return oldValue;
}
```

**步骤1：**segment尝试获取锁， 如果失败就调用**scanAndLockForPut**获得创建的HashEntry

**步骤2：**根据哈希值定位键值对在HashEntry数组上的位置

**步骤3：**遍历链表，如果对应的键已经存在，则直接覆盖原值

**步骤4：**如果node不为null的情况下，把新插入的node节点作为链表的第一个节点，否则就创建一个HashEntry同样把它设置为第一个节点

**步骤5：**容量加一，进行扩容判断，扩容的条件是：当前的容量大于阀值并且小于最大容量，则进行**rehash**扩容操作，然后tab数组index位置的首节点设置为node

**存疑：**决定node是否为空的依据就是tryLock是否成功，成功了就会延时创建，不成功则会调用**scanAndLockForPut**创建，这个方法做了什么呢？其次，扩容机制是怎么实现的呢？

## [Segment-> scanAndLockForPut]

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        // 1
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; // negative while locating node
        // 2 
        while (!tryLock()) {
            HashEntry<K,V> f; // to recheck first below
            // 2.1
            if (retries < 0) {
                if (e == null) {
                    if (node == null) // speculatively create node
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            // 2.2
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            // 2.3
            else if ((retries & 1) == 0 &&
                    (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
```

**步骤1：**根据hash值获取tab数组中对应的entry

**步骤2：**CAS方式获得锁，在这个过程中：

* **2.1：** **retries**自选次数的初值为-1，这里依然会遍历链表，如果当链表头为空或者key已经存在的情况下会把**retries**的值置为0，即代表可以开始自旋
* **2.2：**当可以开始自旋之后每次自选会递增**retries**的值，直到到达最大的自选次数，而这个最大值是由cpu的数量来决定的，如果是在多cpu的情况下则允许64次的最大自旋，到达了便阻塞，用阻塞的方式解决自旋带来的性能开销，而如果是单cpu的情况下，则最大自旋的次数只有1
* **2.3：**当自旋过程中头节点发生变化，则给**retries**重新赋值为-1，也就是会对链表重新遍历 

**小结：**这个方法的实现非常厉害，如果在**segment-> put**操作的开始能够获得锁那当然是最好的，但是即便没拿到，它也不会傻等，或者马上就阻塞，而是会以乐观锁的方式架设链表不会变，然后如果链表变了再重新遍历

## [Segment-> rehash]

```java
 private void rehash(HashEntry<K,V> node) {
     // 1
     HashEntry<K,V>[] oldTable = table;
     int oldCapacity = oldTable.length;
     int newCapacity = oldCapacity << 1;
     threshold = (int)(newCapacity * loadFactor);
     HashEntry<K,V>[] newTable =
                (HashEntry<K,V>[]) new HashEntry[newCapacity];
     int sizeMask = newCapacity - 1;      
     for (int i = 0; i < oldCapacity ; i++) {
         // 2
         HashEntry<K,V> e = oldTable[i];
        if (e != null) {
          HashEntry<K,V> next = e.next;
          int idx = e.hash & sizeMask;
          if (next == null)   //  Single node on list
              newTable[idx] = e;         
          } else { // Reuse consecutive sequence at same slot
              HashEntry<K,V> lastRun = e;
              int lastIdx = idx;
              for (HashEntry<K,V> last = next;
                   last != null;
                   last = last.next) {
                   int k = last.hash & sizeMask;
                   if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                   }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                     V v = p.value;
                     int h = p.hash;
                     int k = h & sizeMask;
                     HashEntry<K,V> n = newTable[k];
                     newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
          } 
     }
      // 3
      int nodeIndex = node.hash & sizeMask; // add the new node
      node.setNext(newTable[nodeIndex]);
      newTable[nodeIndex] = node;
      table = newTable;
 }
```

**步骤1：**获取旧的容量并且计算新的容量，新的容量是旧容量的两倍，然后便创建新的数组和新的掩码

**步骤2：**遍历旧的tab数组并用hash值计算新的位置，如果旧tab数组中当前位置的元素只有一个，那么直接赋值到新数组就可以了，如果不只有一个元素，那就一直往后遍历，直到找到和头结点不同索引值，那么那之后节点就在另一个桶里面了，而剩下的就是从头开始遍历到不相同节点的前一个节点，分别把这两条链赋值到新数组相应的位置就行了

**步骤3：**收尾操作：处理引起扩容的那个待添加的节点并且把新tab数组赋值上去

## [ConcurrentHashMap-> get]

```java
 public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        // 1
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 2
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

**步骤1：**计算相应的hash值，然后与**put**操作类似，然后根据hash值计算key在哪个segment ，然后可见性获取segment

**步骤2：**获取segment中的tab数组，遍历table中的HashEntry元素，找到相同的key，返回value  

**小结：**读操作之所以这么轻松简单，得益于写操作的贡献，由于写操作会把新节点添加在链表头，所以不会影响读的操作，而由于有**UNSAFE** api，保证了能够无锁且获取到最新的volatile变量的值 

## [ConcurrentHashMap-> remove]

```java
public V remove(Object key) {
        int hash = hash(key);
        Segment<K,V> s = segmentForHash(hash);
        return s == null ? null : s.remove(key, hash, null);
 }
```

调用segment的remove方法执行删除

## [Segment-> remove]

```java
final V remove(Object key, int hash, Object value) {
    // 1
    if (!tryLock())
            scanAndLock(key, hash);    
     V oldValue = null;
     try {
         // 2
         HashEntry<K,V>[] tab = table;
         int index = (tab.length - 1) & hash;
         HashEntry<K,V> e = entryAt(tab, index);   
         HashEntry<K,V> pred = null;
         // 3
         while (e != null) {
              K k;
              HashEntry<K,V> next = e.next;  
              if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                    V v = e.value;
                    if (value == null || value == v || value.equals(v)) {
                        if (pred == null)
                            setEntryAt(tab, index, next);
                        else
                            pred.setNext(next);
                        ++modCount;
                        --count;
                        oldValue = v;
                    }
                    break;
                }
              pred = e;
              e = next;
          }
     } finally {
        unlock();
     }
     return oldValue;
}
```

**步骤1：**与添加逻辑相似，尝试获得锁，如果失败了就调用**scanAndLock**方法

**步骤2：**依然是根据哈希值定位到对应tab数组的位置，并创建一个pred节点，应该是用来保存待删除节点的前一个，继续往下看

**步骤3：**遍历链表，如果找到相同的key而且从该键取出来的值又等于传入的value，这个时候就可以执行删除操作，如果待删除节点就是链表的头结点，那么数组该位置上的元素就换成待删除节点的后一个，否则，就把连接待删除节点的prev和next，完成节点的删除

## [Segment-> scanAndLock]

```java
 private void scanAndLock(Object key, int hash) {
        // 1   
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        int retries = -1;
        // 2
        while (!tryLock()) {
            HashEntry<K,V> f;
            if (retries < 0) {
               if (e == null || key.equals(e.key))
                   retries = 0;
                else
                   e = e.next;
            }
            else if (++retries > MAX_SCAN_RETRIES) {
                 lock();
                 break;
             }
             else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f;
                    retries = -1;
            }
        }
   }
```

**步骤1：**依然是从tab数组中找到该hash值的第一个元素，也就是链表头，然后同样有一个**retries**自旋次数

**步骤2：**这里执行的操作和**put**操作比较相似，只不过这里的条件变成链表为空以及找到了对应的节点，这里就没有可以在CAS中创建节点的操作了



参考资料：

[Java泛型底层源码解析](https://www.cnblogs.com/liang1101/p/6407871.html)

[多线程十一之ConcurrentHashMap](https://www.cnblogs.com/rain4j/p/10972090.html)