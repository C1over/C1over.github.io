---
layout:     post   				    
title:      数据结构回顾(五)ConcurrentHashMap1.8
subtitle:   数据结构回顾系列   #副标题
date:       2019-9-26		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-re-vs-ng2.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 数据结构回顾系列
---

# 数据结构回顾(五)ConcurrentHashMap1.8

## 前言

上一篇文章回顾了1.7版本下ConcurrentHashMap的实现，这一篇文章就是学习并记录下ConcurrentHashMap1.8的实现，分开两篇文章来写主要是还是想curd都走一遍，而且不让一篇文章太累赘

## Field

```java
// 默认为0，用来控制table的初始化和扩容操作
private transient volatile int sizeCtl;
// 默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方
transient volatile Node<K,V>[] table;
// 默认为null，扩容时新生成的数组，其大小为原数组的两倍
private transient volatile Node<K,V>[] nextTable;
```

## 构造函数

```java
/**
 * Creates a new, empty map with the default initial table size (16)
 */
public ConcurrentHashMap() {
}
```

```java
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```

从构造函数和成员变量可以看出来，在1.8版本，ConcurrentHashMap消除了segment的概念，如果不传入初始化的容量，那么默认为16，如果传入了初始化的容量，那么会调用**tableSizeFor**方法返回大于输入参数且最近的2的整数次幂的数，和**HashMap**的设定差不多

## [ConcurrentHashMap-> put]

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

## [ConcurrentHashMap-> putVal]

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 1 
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;   
    for (Node<K,V>[] tab = table;;) {
        // 2
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
                tab = initTable();    
        // 3
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
        }
        // 4
        else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 5
            synchronized (f) {
               // 5.1
               if (tabAt(tab, i) == f) {
                      if (fh >= 0) {
                          binCount = 1;
                          for (Node<K,V> e = f;; ++binCount) {
                              K ek;
                              if (e.hash == hash &&  
                                  ((ek = e.key) == key ||
                                  (ek != null && key.equals(ek)))) {
                                   oldVal = e.val;
                                   if (!onlyIfAbsent)
                                        e.val = value;
                                   break;
                               }
                               Node<K,V> pred = e;
                               if ((e = e.next) == null) { 
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                               }
                           }
                      }
                      // 5.2
                      else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                           }
                      }
                    // 6
                    if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
    }
    // 7 
    addCount(1L, binCount);
    return null;
}
```

**步骤1：**对key与value的判空以及调用**spread**方法计算出hash值

**步骤2：**对table数组的延时加载，如果还未创建，则调用**initTable**创建table数组

**步骤3：**根据步骤1计算出来的哈希值，运用取余的操作计算索引，如果当前tab数组中索引位置的项为空，那就以CAS的方式插入进去，值得注意的其实是**tabAt**和**casTabAt**方法其实和1.7的设计是一样，还是会用UNSAFE进行内存操作

**步骤4：**如果当前ConcurrentHashMap正在扩容，先协助扩容，再插入节点

**步骤5：**来到步骤5的这个else里面，其实就是hash冲突的情况了，会把tab[i]锁起来

* **5.1：**这种情况下处理的是链表，处理的流程和HashMap类似，遍历链表，如果遇到了相同的key，就把value覆盖进去，如果节点不存在则添加到链表的尾端
* **5.2：**如果是红黑树节点则用红黑树的方式添加，操作和HashMap基本一致

**步骤6：**根据链表长度判断是否需要把链表转换成红黑树

**步骤7：**统计节点个数，判断是否需要resize

## [ConcurrentHashMap-> tabAt]

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
      return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

## [ConcurrentHashMap-> casTabAt]

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

## [ConcurrentHashMap-> spread]

```java
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
}
```

计算hash值的操作和HashMap类似，但是这里除了有哈希值高低位异或之外还会与上一个BITS

## [ConcurrentHashMap-> helpTransfer]

## [ConcurrentHashMap-> transfer]

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // 1
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 2
    if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
     }
   
     for (int i = 0, bound = 0;;) {
        // 3
        Node<K,V> f; int fh;
        while (advance) {
          int nextIndex, nextBound;
          if (--i >= bound || finishing)
               advance = false;
          else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
           }
           else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                 nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                 bound = nextBound;
                 i = nextIndex - 1;
                 advance = false;
            }
     }
     // 4
     if (i < 0 || i >= n || i + n >= nextn) {
          int sc;
          if (finishing) {        
               nextTable = null;
               table = nextTab;
               sizeCtl = (n << 1) - (n >>> 1);   
               return;
          }
          if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
          } 
       }  
       // 5
       else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
       // 6
       else if ((fh = f.hash) == MOVED)
                advance = true; // already processed  
       // 7
       else {
          synchronized (f) {
             if (tabAt(tab, i) == f) {
                 Node<K,V> ln, hn;
                 if (fh >= 0) {
                   // 7.1  
                   int runBit = fh & n;
                   Node<K,V> lastRun = f;
                   for (Node<K,V> p = f.next; p != null; p = p.next) {
                      int b = p.hash & n;
                      if (b != runBit) {
                          runBit = b;
                          lastRun = p;
                       }
                    }
                    if (runBit == 0) {
                         ln = lastRun;
                         hn = null;
                     }
                     else {
                         hn = lastRun;
                         ln = null;
                     }   
                     // 7.3
                     for (Node<K,V> p = f; p != lastRun; p = p.next) {
                         int ph = p.hash; K pk = p.key; V pv = p.val;
                         if ((ph & n) == 0)
                             ln = new Node<K,V>(ph, pk, pv, ln);
                         else
                             hn = new Node<K,V>(ph, pk, pv, hn);
                       }
                       setTabAt(nextTab, i, ln);
                       setTabAt(nextTab, i + n, hn);
                       setTabAt(tab, i, fwd);
                       advance = true;
                }            
             }  
             // 8 
             else if (f instanceof TreeBin) {
                      TreeBin<K,V> t = (TreeBin<K,V>)f;
                      TreeNode<K,V> lo = null, loTail = null;
                      TreeNode<K,V> hi = null, hiTail = null;
                      int lc = 0, hc = 0;
                      for (Node<K,V> e = t.first; e != null; e = e.next) {
                         int h = e.hash;
                         TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                         if ((h & n) == 0) {
                            if ((p.prev = loTail) == null)
                                    lo = p;
                             else
                                    loTail.next = p;
                                 loTail = p;
                                 ++lc;
                          }
                          else {
                             if ((p.prev = hiTail) == null)
                                    hi = p;
                             else
                                     hiTail.next = p;
                                 hiTail = p;
                                 ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                } 
          }       
       }  
    }
}
```

**步骤1：**根据旧的容量以及CPU个数计算**stride**值，逻辑就是如果CPU个数大于1，就把旧容器的长度除以8，然后还有个最小的边界16，如果是单CPU就不做处理

**步骤2：**在这一步中会创建容量为旧容量2倍的新数组，然后保存在**nextTable**成员中，并且用成员**transferIndex**记录下旧容量

**步骤3：**这一部分暂时没理解 // todo

**步骤4：**表示完成转移，完成赋值，并且**sizeCtl**赋值为新容量的0.75倍

**步骤5：**数组中把null的元素设置为ForwardingNode节点(hash值为MOVED[-1]

**步骤6：**判断数组中的元素是否替换为ForwardingNode节点

**步骤7：**进入这里之后就会给对应的节点进行加锁，在此之后会判断一下hash值，如果大于等于0说明是正常的节点，不然就不用操作了

* **7.1：**这个计算的思路和HashMap是一致的，由于长度是2的幂指，所以只需要判断，多出来的那一个位是是1还是0就可以判断出扩容之后这个链表是放在原来的位置还是原来位置+旧容量的地方了
* **7.2：**然后遍历操作其实和1.7类似，就是找到变化的最后一个节点
* **7.3：**而剩下的操作就是把两条链表赋值到新数组对应的位置以及把原来的数组赋上fwd，相比于**HashMap**构建新链表时每个节点都要处理的操作，ConcurrentHashMap添加了最后一个变化节点这种概念去进行优化，然最后变化节点后续的节点不需一个个处理

**步骤8：**步骤8与步骤7类似，只是针对红黑树进行操作，除此之外在复制完树节点之后，判断该节点处构成的树还有几个节点，如果≤6个的话，就转回为一个链表

## [Summary]

扩容过程有点复杂，稍微总结一下扩容过程的流程：

* 扩容操作中，会找到最后一个位变化的节点，然后把原本的链表切割为两个链表分别赋值在原位和原位+旧容量的位置
* 扩容操作中，会把旧数组中对应的位置的节点置为ForwardingNode节点(hash值为MOVED[-1]，标记这个节点为空
* 而在完成状态下，几个重要成员变量的值变化如下：
  * transferIndex：旧容量
  * sizeCtl：新容量的0.75倍

纵观整个**put-> helpTransfer-> transfer**这个过程：

* 如果检测到了tab数组中有置为ForwardingNode节点(hash值为MOVED[-1]的节点，说明正在扩容
* 这个时候1.8ConcurrentHashMap做的优化处理是让当前线程也会参与去复制，通过允许多线程复制的功能，以此来减少数组的复制所带来的性能损失

## [ConcurrentHashMap-> get]

```java
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 1
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            // 2
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 3
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
}
```

**步骤1：**与put操作相似的哈希值的计算

**步骤2：**计算key的hash值来定位元素在数组中的位置

**步骤3：**遍历链表，找相同key，获得value

## [Summary]

在1.8版的ConcurrentHashMap中消除了segment的概念，同步处理主要是通过synchronized和UNSAFE，在取得sizeCtl、某个位置的Node的时候，使用的都是unsafe的方法，来达到并发安全的目的，当需要在某个位置设置节点的时候，则会通过synchronized的同步机制来锁定该位置的节点，而其实除了这点区别外，1.8相比1.7ConcurrentHashMap在插入时的优化方案相对不一，1.7版本会合理利用自旋的时候创建节点，而由于1.8版本用的是synchronize，失去了部分灵活性，但是设计者又会在获得锁之前判断是否在扩容，如果在扩容则让该线程也参与



参考资料：

[ConcurrentHashMap](https://www.cnblogs.com/zerotomax/p/8687425.html)





