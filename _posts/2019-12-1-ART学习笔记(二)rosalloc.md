---
layout:     post   				    
title:      ART学习笔记(一)Rosalloc
subtitle:   ART学习笔记   #副标题
date:       2019-12-2		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-universe.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - ART学习笔记
---

# ART学习笔记(二)Rosalloc

## 前言

上一篇笔记记录了**ART**虚拟机中的各种**Space**，而**MallocSpace**这个虚类就是**ART**设计与实现如C语言般的内存分配及释放，而且**ART**中默认用的是**RosAllocSpace**，它的内存分配和释放算法是依赖于rosalloc实现的

## # RosAlloc关键类及成员变量

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/rosalloc2.jpg)

* **FreePageRun：** 在RosAlloc中通过AllocPages提供整数倍页大小内存分配的接口，而**FreePageRun**是RosAlloc中的内部类，用于帮助RosAlloc管理页内存的分配
* **Slot：**代表内存分配的单元（有42种不同的粒度）
* **Run：** **RosAlloc**的内部类，一个**Run**对象代表了一个内存分配的资源池（多个内存分配粒度一样的**slot**）
* **SlotFreeList：**管理**Slot**的空闲链表
* **RosAlloc：**实际提供内存分配的关键类

```c++
// RosAlloc成员变量声明
// kNumOfSizeBrackets = 42
static size_t bracketSizes[kNumOfSizeBrackets];
static size_t numOfPages[kNumOfSizeBrackets];
```

![](https://github.com/Cc1over/Cc1over.github.io/blob/master/img/rosalloc1.jpg?raw=true)

* **bracketSizes：** **rosalloc**设计了42种不同粒度的内存分配单元（Slot），**bracketSizes**用于描述每种slot所支持的内存分配粒度，比如：
  * barcketSizes[0] = 8
  * barcketSizes[1] = 16
  * barcketSizes[41] = 2KB
  * 内存分配时要选择一种粒度的**slot**，通常以向上取整的方式选择
* **numOfPages：**记录内中**slot**对应的内存资源又多少（以4KB为单位）
  * 粒度为1KB的**slot**拥有2*4KB内存资源
  * 粒度为2Kb的**slot**拥有4*4KB内存资源
  * 其余粒度的**slot**拥有1*4KB内存资源

## # Run & slot

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/rosalloc3.jpg)

* **Run**对象代表一个内存分配的资源池，它将多个**slot**组织起来，通过**size_bracket_idx_**可以获取**Run**相关的各种信息：
  * **bracketSizes[size_bracket_idx_]** = **Run**中**slot**内存分配的粒度
  * **numOfPages[size_bracket_idx_]** = **Run**中拥有的内存资源大小
  * **headerSizes[size_bracket_idx_]** = **Run**中整个头部空间的大小
  * **numOfSlots[size_bracket_idx_]** = **Run**中**slot**的个数

## # RosAlloc中的内存划分

```c++
// RosAlloc成员变量声明
// 代表rosalloc要管理的那块内存
uint8_t* base_;
// 这块内存资源目前的大小 <= max_capacity
size_t capacity_;
// 这块内存资源的最大尺寸
size_t max_capacity_;
// 用于保存base_内存分配情况以及一些状态
std::unique_ptr<MemMap> page_map_mem_map;
// 内存映射对象对应内存基地址
volatile unit8_t* page_map_;
size_t page_map_size_;
size_t max_page_map_size_;
```

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/rosalloc4.jpg)

* 下半部分：base_代表**rosalloc**中所管理的内存资源：
  * 以 4KB/页 为单元进行划分
  * 内存资源最大为max_capacity
  * 内存页个数为page_map_size_
  * base_可强转成**FreePageRun**
* 上半部分：用于记录内存页的状态，与下半部分的内存页一一对应，数组中每一个元素保存base_中每一页的信息

## # RosAlloc-> Alloc

## [rosalloc-inl.h-> RosAlloc::Alloc]

```c++
inline ALWAYS_INLINE void* RosAlloc::Alloc(Thread* self, size_t size, size_t* bytes_allocated, size_t* usable_size, size_t* bytes_tl_bulk_allocated) {
    if(size > 2KB) {
        return AllocLargeObject(self,size,bytes_allocated,usable_size,bytes_tl_bulk_allocated);
    }else{
        return AllocFromRun(self,size,bytes_allocated,usable_size,bytes_tl_bulk_allocated);
    }
}
```

## [rosalloc-inl.h-> RosAlloc::AllocFromRun]

```c++
void* RosAlloc::AllocFromRun(Thread* self, size_t size, size_t* bytes_allocated, size_t* usable_size, size_t* bytes_tl_bulk_allocated) { 
    // 根据size向上取整决定应该采用哪种粒度的slot
    size idx = SizeToIndexAndBracketSize(size,...);
    if(idx < 16) {
        // bracketSizes[16] = 128, 如果分配的内存不超过128字节，则从线程本地资源池中分配内存
        // 线程本地资源池 ≠ TLAB，但与TLAB类似，是Thread对rosalloc单独支持
        // 与TLAB类似，本地资源池的信息存储在tlsPtr_中
        Run* thread_local_run = reinterpret_cast<Run*>(self->GetRosAllocRun(idx));
        slot_addr = thread_local_run->AllocSlot();
        // thread_local_run内存不足
        if(slot_addr == nullptr) {
           MutexLock mu(self,*size_bracket_lock_[idx]);
           // 空闲链表合并 
           if(thread_local_run->MergeThreadLocalFreeListToFreeList) {
               // ......
           } else {
               thread_local_run = RefillRun(self,idx);
               thread_local_run->SetIsThreadLocal(true);
               self->SetRosAllocRun(idx, thread_local_run);
           }
           // 重新内存分配
           slot_addr = thread_locak_run->AllocSlot();
        }
    } else {
        // 当所需内存超过128字节时，将从RosAolloc内部的资源池进行分配，这个时候需要同步锁保护
        // 不同大小的资源池使用不同的同步锁保护
        MutexLock mu(self, *size_bracket_locks_[idx]);
        slot_addr = AllocFromCurrentRunUnlocked(self, idx);
        // 一些校验判空
    }
    return slot_addr;
}
```

* 当分配内存不超过128字节，则从线程本地资源池中分配内存
* 而当在线程本地资源池中内存不足的情况下：
  * 把thread_local_free_list_中的空闲资源合并到Run中free_list_中
  * 调用**RefillRun**重新分配一个Run对象
  * 把这个新Run对象设置为thread_local_run
  * 在这个新的Run中分配内存
* 当所需内存超过128字节，将从RosAlloc内部的资源池进行分配，不同大小的资源池使用不同的同步锁

## [rosalloc.cc->RosAlloc::RefillRun]

```c++
RosAlloc::Run* RosAlloc::AllocRun(Thread* self,size_t idx) {
    RosAlloc::Run* new_run = nullptr;
    MutexLock mu(self,lock_);
    // 从base_所在的内存中分配一段内存空间，这段内存空间由一个Run对象进行管理
    // 这段空间的大小为numOfPage[idx]
    // kPageMapRun代表内存状态的一种
    new_run = reinterpret_cast<Run*>(AllocPages(self,numOfPage[idx],kPageMapRun));
    if(new_run!=nullptr) {
        // 初始化Run中的空闲链表
        new_run->InitFreeList();
    }
    return new_run;
}
```

* 从base_中以页为单位分配Run，页数为numOfPage[idx]的值
* 初始化Run中的Slot空闲链表

## [rosalloc.h->RosAlloc::Run::InitFreeList]

```c++
void initFreeList() { 
   const uint8_t idx = size_bracket_idx_;
   const size_t bracket_size = brackerSizes[idx];
   Slot* first_slot = FirstSlot();
   for(Slot* slot = LastSlot();slot>=first_slot;slot=slot->Left(bracket_size)) {
      free_list_.Add(slot);     
   } 
}
```

## [rosalloc-inl.h->RosAlloc::Run::AllocSlot]

```c++
inline void* RosAlloc::Run::AllocSlot(){
    Slot* slot = free_list_.Remove();
    return slot;
}
```

* Run是从base_中以整页为单位分配过来的，而在Run的初始化过程中会初始Run中的Slot空闲链表
* Slot空闲链表会按照Slot在Run中的内存布局的位置由前到后排列，而运行过程中考虑到内存释放，Slot空闲链表结构无法保持initFreeList时候的所创建的初始顺序，单不英雄内存分配



**参考资料：** 《深入理解Android虚拟机ART》 