---
layout:     post   				    
title:      I/O回顾系列(一)Buffer I/O
subtitle:   I/O回顾系列   #副标题
date:       2019-10-1		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-swift.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - I/O回顾系列
---

# I/O回顾系列(一)Buffer I/O

## # 前言

上一个数据结构回顾系列中回顾了一些android中以及java中常见的一些数据结构的实现，而最近终于有空可以抽出时间回顾下一个专题，这个专题其实早在apk学习系列后就想开启，因为当时不管是看**Walle**也好，**VasDolly**也好，其实它们内部都用到了NIO，不过无奈课真的太多了......，现在这个回顾系列会从Buffer I/O -> OK IO -> NIO

## # Constructor

```java
public BufferedInputStream(InputStream in) {
        this(in, DEFAULT_BUFFER_SIZE);
}
```

```java
public BufferedInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
 }
```

在构造函数里会创建一个字节数组，默认的长度为8192

## # Field

```java
private static int DEFAULT_BUFFER_SIZE = 8192; // 默认的缓冲区存储数据大小
protected volatile byte buf[]; // 缓存区字节数组
protected int count; // 当前缓冲区有效数据
protected int pos; // 当前缓冲区读的索引
protected int markpos = -1; // 当前缓冲区的标记位置
protected int marklimit     // 缓冲区可标记位置的最大值
private static int MAX_BUFFER_SIZE = Integer.MAX_VALUE - 8;  // 缓冲区最大容量
```

## # BufferInputStream-> read

```java
public synchronized int read() throws IOException { 
    if (pos >= count) {
         fill();
         if (pos >= count)
                return -1;
    }
    return getBufIfOpen()[pos++] & 0xff;
}
```

read方法读取的是一个字节，可以看到方法的逻辑是判断当前可读的位置是否大于缓冲区的有效数组大小，如果是就调用**fill**方法，然后再判断一次，如果结果依旧就返回-1代表读完了，否则则调用**getBufIfOpen**返回一个字节，并且有个小操作就是会&上0xff，这是因为返回值的类型是int类型，而读取出来的只有一个字节，&0xff的目的其实就是为了保证二进制补码的一致性，当该字节是负数的时候将高位清除 

## # BufferInputStream-> getIfOpen

```java
private byte[] getBufIfOpen() throws IOException {
        byte[] buffer = buf;
        if (buffer == null)
            throw new IOException("Stream closed");
        return buffer;
}
```

该方法逻辑十分简单，其实就是判断当前的流有没有被关闭，没有就返回缓冲区字节数组

## # BufferInputStream-> mark

```java
public synchronized void mark(int readlimit) {
        marklimit = readlimit;
        markpos = pos;
}
```

按照源码的思路，我们下一个要看的方法应该是**fill**，那为什么看**mark**方法呢？，因为在**fill**方法中十分多对markpos和marklimit值的判断和使用，而整个类中对markpos和marklimit赋非特殊值的只有**mark**方法

## # BufferInputStream-> fill

```java
private void fill() throws IOException {
    // 1    
    byte[] buffer = getBufIfOpen();
    if (markpos < 0）  pos = 0;   
    // 2
    else if (pos >= buffer.length)  /* no room left in buffer */
       // 2.1
       if (markpos > 0) {  /* can throw away early part of the buffer */
          int sz = pos - markpos;
          System.arraycopy(buffer, markpos, buffer, 0, sz);
          pos = sz;
          markpos = 0; 
       }
       // 2.2 
       else if (buffer.length >= marklimit) {
           markpos = -1;   /* buffer got too big, invalidate mark */
           pos = 0;        /* drop buffer contents */
       }
       // 2.3 
       else if (buffer.length >= MAX_BUFFER_SIZE) {
           throw new OutOfMemoryError("Required array size too large");
       }
       // 2.4
       else {
           int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                        pos * 2 : MAX_BUFFER_SIZE;
           if (nsz > marklimit)
                    nsz = marklimit;
           byte nbuf[] = new byte[nsz];
           System.arraycopy(buffer, 0, nbuf, 0, pos);
           if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
              throw new IOException("Stream closed");
           }
           buffer = nbuf;
       }
      // 3  
      count = pos;
      int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
      if (n > 0)
            count = n + pos;    
}
```

**步骤1：**获取缓存数组并判断markpos是否为-1，如果为-1的话就把pos置为0

**步骤2：**如果此时pos大于buffer的长度，也就是读到末尾的时候：

* **2.1：**当markpos大于零，将markpos ~ pos的数据左移，然后pos置在左移数据区的末尾，markpos就置在开头，也就是说数据左移后移动完后数据位于 0 ~ sz，移动完后目前sz之后的数据无效
* **2.2：**当缓冲区容量大于可标记的最大值，把markpos置为-1，pos置为0，也就是说缓冲区容量大于可标记限制，所有数据都不要了，markpos标记也不要了
* **2.3：**当缓冲区容量过大，抛出异常
* **2.4：**当可标记区域大于缓冲区容量，对缓冲区进行扩容，但是这里有一个比较细节的点，那就是其实**bufUpdater**  CAS操作的并不是实际的内存只是引用，这样做可以减少内存的拷贝过程，所以最后才需要把buffer的引用也改成nbuf

**步骤3：**首先把count记录为当前pos的位置，然后使用in读取，把读取的数据存储在buffer中，然后重新计算长度

## # Summary

整个read流程就走完了，所以其实BufferInputStream所做的其实就是把in中的数据读到一个缓冲区中，如果pos没有到末尾就取出，如果到末尾了，会尝试更新缓冲区，更新完后如果还是末尾，就说明读完了，如果能读就读

## # BufferInputStream-> read

```java
public synchronized int read(byte b[], int off, int len)
        throws IOException
    {    
        // 1
        getBufIfOpen(); // Check for closed stream
        if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }
        // 2
        int n = 0;
        for (;;) {
            int nread = read1(b, off + n, len - n);
            if (nread <= 0)
                return (n == 0) ? nread : n;
            n += nread;
            if (n >= len)
                return n;
            // 3
            InputStream input = in;
            if (input != null && input.available() <= 0)
                return n;
        }
    }
```

**步骤1：**获取buf，然后检查要获取的数据（假设有），b[]是否内存得下

**步骤2：**读取数据到b[], nread为代表读取的数量，然后调用**read1**执行读操作，最后再计算读取的数据量

**步骤3：**判断in是否已经被关闭并且再没有可读数据，就直接返回了

## # BufferInputStream-> read1

```java
private int read1(byte[] b, int off, int len) throws IOException {
        // 1
        int avail = count - pos;
        if (avail <= 0) {
            // 1.1
            if (len >= getBufIfOpen().length && markpos < 0) {
                return getInIfOpen().read(b, off, len);
            }
            // 1.2
            fill();
            avail = count - pos;
            if (avail <= 0) return -1;
        }
        // 2
        int cnt = (avail < len) ? avail : len;
        System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
        pos += cnt;
        return cnt;
    }
```

**步骤1：** 计算缓冲区可读区域的大小，如果已经没可读数据：

* **1.1：**判断需要读取的数据量大于缓冲区能读取的大小，直接交给in去读取
* **1.2：**缓冲区已没有可读取的数据，对缓冲区填充，并记录缓冲区有效数据，最后再次判断有无可读数据就可以了

**步骤2：**如果有可读数据，计算将要读入 b[] 的数据量，然后将将缓冲区的数据读入 b[]，最后更新缓冲区索引位置

## # BufferOutStream

```java
public synchronized void write(int b) throws IOException {
        if (count >= buf.length) {
            flushBuffer();
        }
        buf[count++] = (byte)b;
}
```

```java
private void flushBuffer() throws IOException {
        if (count > 0) {
            out.write(buf, 0, count);
            count = 0;
        }
 }
```

而写的操作相对更加简单，只需要先写到缓存中，缓存满了，调用out一次性写入就可以了

## # Summary

Buffer I/O的优化得益于缓冲区的存在，减少了系统调用。也就是说，如果缓冲区能满足读入/写出需求，则不需要进行系统调用，维护系统读写数据的习惯 







参考资料：

[Okio好在哪](https://www.jianshu.com/p/2fff6fe403dd)