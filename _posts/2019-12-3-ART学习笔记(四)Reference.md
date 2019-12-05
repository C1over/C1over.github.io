---
layout:     post   				    
title:      ART学习笔记(四)Reference
subtitle:   ART学习笔记   #副标题
date:       2019-12-3		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-universe.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - ART学习笔记
---

# ART学习笔记(四)Reference

## # Reference关键类

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/reference1.jpg)

**Reference**是一个抽象模但是其中没有声明**abstract**方法，但是**Reference**有四个派生类，分别是**SoftReference**、**WeakReference**、**PhantomReference**和**FinalizerReference**

假如存在一个非**Reference**对象obj，对GC可能有三种情况：

* 通过非**Reference**的引用能搜索到obj，这种情况下，obj不会被回收
* 通过**Reference**的引用搜索到obj，这种情况下，obj是否被释放取决于它的那个**Reference**对象的数据类型以及此次GC的回收策略
* 无法搜索到的obj，obj会被回收

以ART虚拟机**CMS**为例：

* refObj的类型为**SoftReference**
  * **StickyMarkSweep**不会释放**SoftReference**所引用的对象
  * **PartialMarkSweep**与**MarkSweep**都会释放**SoftReference**所引用的对象
* refObj的类型为**WeakReference** 
  * 不论回收策略是什么，**WeakReference**所引用的对象都会被释放
* refObj的类型为**PhantomReference**
  * **PhantomReference**必须配合**ReferenceQueue**使用
  * 不论回收策略是什么，**PhantomReference**所引用的对象都会被加到**ReferenceQueue**中，但是**PhantomReference**所引用的对象不会被回收，只有从**ReferenceQueue**中取出来，才会在下一次GC过程中被回收

## # Reference关键成员变量及函数

```java
public abstract class Reference<T> {
    private static boolean slowPathEnable = false;
    // referent指向的实际对象
    volatile T referent;
    // 关联的ReferenceQueue
    final ReferentQueue<? super T> queue;
    // 多个Reference对象借助queueNext组成一个单向链表
    Reference queueNext;
    // pendingNext也用于将多个Reference对象组成一个单向链表
    // 这个链表的用法主要和GC有关
    Reference<?> pendingNext;
    // 获取所指向的实际对象
    public T get() {
        return getReference();
    }

    private final native T getReferent();
    
    // 解绑和实际对象的关联
    public void clear() {
       this.referent = null;
    }
    
}
```

## # GC中对Reference的处理

**以MarkSweep为例**，GC中对**Reference**进行处理的时机为：

* 调用**InitializePhase**执行GC初始化阶段时，在这个阶段中设置**clear_soft_reference_**成员变量，也就是设置是否释放**SoftReference**所引用对象的标记为，而对**CMS**而言，只有在**kGcTypeSticky**也就是**StickyMarkSweep**的时候，这个标记为会为false，其他情况都为true
* 调用**PausePhase**函数的过程中，把**Reference**类的静态变量**slowPathEnabled**置为true
* 当每找到一个对象，则会进行判断这个obj是否是一个**Reference**对象，如果是则调用**ReferenceProcessor**的**DelayReferenceReferent**函数
* 调用**ReclaimPhase**执行GC回收阶段时，将会调用**ReferenceProcessor**的**ProcessReferences**函数，而MS中**clear_soft_reference**成员变量也在这个时候作为这个方法的参数而起作用
* **ReferenceProcessor**是在**Heap**中创建的，用于Reference的处理

### [reference_processor.cc->DelayReferenceReferent]

```c++
void ReferenceProcessor::DelayReferenceReferent(mirror::Class* kclass, mirror::Reference* ref, collector::GarbageCollector* collector) {
    // 获取Reference所指向的实际对象
    mirror::HeapReference<mirror::Object>* referent = ref->GetReferentReferenceAddr();
    Thread* self = CurrThread();
    if(referent->AsMirrorPtr != nullptr && !collector->IsMarkedHeapReference(referent)) {
        // kclass为ref所属的类型
        // 以下代码就是根据ref类型kclass加到不同ReferenceQueue中
        // ReferenceQueue中的Reference将通过pendingNext构成一个单向链表 
       if(kclass->IsSoftReferenceClass()) {
          soft_reference_queue_.AtomicEnqueueIfNotEnqueued(self,ref);    
       } else if(kclass->IsWeakReferenceClass()) {
          weak_reference_queue_.AtomicEnqueueIfNotEnqueued(self,ref);    
       } else if(kclass->IsFinalizerReferenceClass()) {
          finalizer_reference_queue_.AtomicEnqueueIfNotEnqueued(self,ref);    
      } else if(kclass->IsPhantomReferenceClass()) {
          phantom_reference_queue_.AtomicEnqueueIfNotEnqueued(self,ref);    
     }
   }     
}
```

* 如果这个Referent对象引用的实际对象有没被MarkSweep标记，就会把Reference对象加到对应的ReferenceQueue中，而Reference对象通过pendingNext成员构成一个单向链表
* 如果这个Referent对象引用的实际对象被MarkSweep标记过，说明这个实际对象不需要处理

### [process_references.cc->ReferencePrcessor::ProcessReferences]

根据方法的参数**clear_soft_reference**判断是否需要处理**SoftReference**，如果为true则调用**soft_reference_queue_.ForwardSoftReference**方法，这个方法的处理逻辑通过**pending_next_**遍历弱引用构成的链表，然后把原来这些没有标记的**reference**对象进行标记

然后会跟根据引用类型做不同的处理：

* **SoftReference：**调用**soft_reference_queue_.ClearWhiteReference**函数
* **WeakReference：**调用**weak_reference_queue_.ClearWhiteReference**函数
* **PhantomReference：**调用**phantom_reference_queue_.ClearWhiteReference**函数
* **FinalizerReference：**调用**finalizer_reference_queue_.EnqueueFinalizerReferences**函数

### [reference_queue.cc->ReferenceQueue::ClearWhiteReferences]

方法执行流程：

* 从Reference的pending_next_链表中取出一个元素记为ref
* 判断ref指向的实际对象是否被垃圾回收器标记过(针对不需要处理的**SoftReference**在**ProcessReferences**函数中已经进行标记)
* 如果没有被标记：则将实际对象和ref解绑
* 最后把ref对象保存到**ReferenceProcessor**的**cleared_references**中，**clear_references**保存的是实际对象被视为垃圾的Reference对象

### [小结]

总结下来其实在GC过程中对Reference的处理为：

* 在**InitializePhase**初始化的过程中根据GC策略决定是否需要回收**SoftReference**所引用的实际对象，只有GC回收策略为**kGcTypeSticky**时才会回收
* 每当找到一个obj对象之后进行判断，如果是Reference类型，则根据实际的kclass添加到对应的**ReferenceQueue**中，根据**pendingNext**串起来构成一个单链表
* 在**ReclaimPhase**执行GC回收阶段的时候如果**Reference**类型不是**FinalizerReference**就会根据**pendingNext**遍历**Reference**单链表，然后判断是否标记，如果没标记就进行解绑并把这个Reference对象添加到**ReferenceProcessor**的**cleared_references**中，如果是**FinalizerReference**则会调用**finalizer_reference_queue_.EnqueueFinalizerReferences**函数
* **注意：**这里提到的**ReferenceQueue**是C++层的**ReferenceQueue**

## # Reference & ReferenceQueue

上文的流程总结了**WeakReference**和**SoftReference**在GC过程中造成的影响，但却没有看到**Reference**和**Java ReferenceQueue**的关联？而谜题的答案其实就在**ReferenceProcessor**的**cleared_references**中

### [Heap::CollectGarbageInternal]

这个函数的主要工作是：

- 找到合适的垃圾回收器对象并完成垃圾回收
- 执行GC后的数据统计，统计情况会影响下次垃圾回收的回收策略
- 调用**reference_proceesor**的**EnqueueClearedReferences**函数

在GC结束后，垃圾对象被回收完了，下一步处理的就是**ReferenceProcessor**的**cleared_references**

### [reference_pocessor.cc->ReferencePocessor::EnqeueClearedReferences]

封装一个**ClearedReferenceTask**并交交由当前线程直接执行

### [heap.cc-> ClearedReferenceTask]

调用**Java ReferenceQueue**的add函数，把**ReferenceProcessor**的**cleared_references**添加进去，明确的是这个**Java ReferenceQueue**并不是**Reference**创建的时候绑定的**ReferenceQueue**

### [ReferenceQueue.java-> Reference::add]

```java
static void add(ReferenceQueue<?> list) {
    synchronized (ReferenceQueue.class) {
        if(unenqeued == null) {
            // unenqueued链表还不存在的情况下
            unenqeued = list;
        }else {
            // 把list加到unenqueue pendingNext所在链表中
        }
        ReferenceQueue.class.notifyAll();
    }
}
```

当**Java ReferenceQueue**把**Reference**对象添加到unenqueued链表中后会唤醒一个线程，它唤醒的线程就是**ReferenceQueueDeamon**线程

### [Daemons.java->ReferenceQueueDaemon]

```java
private static class ReferenceQueueDaemon extends Daemon {
    public void run() {
        while(isRunning()) {
            Reference<?> list;
            synchronized (ReferenceQueue.class) {
                while(ReferenceQueue.unenqueued == null) {
                    ReferenceQueue.class.wait();
                }
                list = ReferenceQueue.unenqueued;
                ReferenceQueue.unenqueued = null;
                ReferenceQueue.enqueuePending(list);
            }    
        }
    }
}
```

### [ReferenceQueue-> enqueuePeding]

```java
 public static void enqueuePending(Reference<?> list) {
        Reference<?> start = list;
        do {
            ReferenceQueue queue = list.queue;
            if (queue == null) {
                // list指向一个Reference对象
                // 一个Reference对象会关联一个ReferenceQueue
                // 取出Reference对象然后把这些Reference对象添加到与之管理的ReferenceQueue中
                Reference<?> next = list.pendingNext;
                list.pendingNext = list;
                list = next;
            } else {
                synchronized (queue.lock) {
                    do {
                        Reference<?> next = list.pendingNext;
                        list.pendingNext = list;
                        queue.enqueueLocked(list);
                        list = next;
                    } while (list != start && list.queue == queue);
                    queue.lock.notifyAll();
                }
            }
        } while (list != start);
    }
```

### [小结]

在ART中其实整套**Reference**处理的流程和Hotspot相差甚远，在ART中对**Reference**添加到**ReferenceQueue**中的流程为：

**C++层：**

* 在GC过程中把解绑的**Reference**添加到**ReferenceProcessor**的**cleared_references**中
* 封装一个**ClearedReferenceTask**并交交由当前线程直接执行
* 调到Java层**ReferenceQueue**静态方法add

**Java层：**

* 把C++层的**cleareed_reference**添加到**ReferenceQueue**的静态变量unenqueued中，并唤醒**ReferenceQueueDeamon**线程
* 一个个取出**Reference**并添加到对应绑定的**ReferenceQueue**中
* 唤醒等待的线程，比如**FinalizerDaemon**线程

## # FinalizerReference

**FinalizerReference**比较特殊，是隐藏类，在Java开发中无法使用，主要为虚拟机内部使用，用于调用对象的**finalize**方法

上文有关**FinalizerReference**的函数调用链：

* **ReclaimPhase**
* **ReferencePrcessor::ProcessReferences**
* **ReferenceQueue.EnqueueFinalizerReferences**

### [reference_queue.cc-> ReferenceQueue::EnqueueFinalizerReference]

函数执行流程：

* 与**SoftReference**以及**WeakReference**类似，从**finalizer_reference_queue_**取出一个**Reference**对象，记为ref
* 与**SoftReference**以及**WeakReference**不同，**FinalizerReference**将主动标记引用的实际对象，因为所引用的实际对象都是定义了finalize函数的对象，这些对象被回收前要调用它们的finalize函数，因此不能再还没有调用finalize函数之前就回收它们
* 在把实际对象与**FinalizerReference**中有一个为**zombie_**的成员变量关联起来
* 解绑ref与实际对象的管理
* 与**SoftReference**以及**WeakReference**类似，将ref保存到**clear_references**中

**FinalizerReferences**关联的是定义了finalize函数的类的实例，由于必须在垃圾回收前调用它们的**finalize**方法，所以这次GC就必须主动标记这些对象，避免这些对象在这次GC中被回收

但是下次垃圾回收还是需要释放这些对象，因此**EnqueueFinalizerReference**函数的目的就是：

* 此次GC主动标记，避免**FinalizerReference**所引用的对象被回收
* 为了在解绑之后仍可调用这些对象的**finalize**方法，把实际对象保存在**FinalizerReference**的**zombie_**成员变量中
* 为了下次GC可以回收这些对象，解除**FinalizerReference**与实际对象解绑

而在**FinalizerReference**与实际对象解绑之后，剩下的线索就是**ReferenceProcessor**的**cleared_references**了，对于**FinalizerReference**来说，其实它也会正常走函数流程：

* **Heap::CollectGarbageInternal**
* **ReferencePocessor::EnqeueClearedReferences**
* **heap.cc-> ClearedReferenceTask**
* **ReferenceQueue.java-> Reference::add**
* **ReferenceQueue-> enqueuePending**

而与之不一样的是，在**ReferenceQueue-> enqueuePending**执行的最后会唤醒等待在**Queue**上的线程，比如**FinalizerDaemon线程**

而当与**FinalizerReference**相关联的**ReferenceQueue**就是**FinalizerReference**中的静态变量**queue**，而当**FinalizerDaemon**线程被唤醒之后，只要不停地从这个静态变量**queue**中取出**FinalizerReference**，然后根据**FinalizerReference**中提前设置好的**zombie**成员变量，就可以找到实际对象，然后调用**finalize**方法

来到这里，其实所有有关**finialize**函数调用谜题都得到了解决，但是**FinalizerReference**是什么时候和这个静态变量**queue**关联起来的呢？

答案其实就是类加载，在类加载过程中如果该类实现了**finalize**函数，就会给这个**kclass**标记上一个**kAccClassIsFinalizable**的标记，而划上这个标记的类在执行**Alloc**方法的时候就会在C++层通过JNI调用到Java层**FinalizerReference**的**add**方法，而就是在这个时候，**FinalizerReference**对象得到创建，并且绑定它自己的静态变量**queue**

## # 总结

* **SoftReference：**不保证每次GC都会回收它们所指向的实际对象，具体到ART虚拟机，回收策略为**kGcTypeSticky**肯定不会回收
* **WeakReference：**每次GC都会回收它们所指向的实际对象
* **PhantomReference：**它的功能与回收没有关系，只是提供一种手段告诉使用者某个实际对象被回收了。使用者可以据此做一些清理工作。目的与finalize类似
* **FinalizerReference：**专门用于调用垃圾对象的finalize函数，finalize函数调用后，垃圾对象会在下一次GC中被回收
* 为什么说**PhantomReference**比finalize函数更优雅，原因为：
  * 虚拟机中只有**FinalizerDaemon**一个线程调用对象的finalize函数，并且，**FinalizerDaemon**是虚拟机提供的，开发者没有办法干预它的工作
  * 如果使用**PhantomReference**的话，开发者就可以根据情况使用多个线程来处理，例如，根据绑定实际对象的类型，通过多个**ReferenceQueue**并使用多个线程来等待他们并处理实际对象被回收后的清理工作















参考资料：《深入理解Android虚拟机ART》 