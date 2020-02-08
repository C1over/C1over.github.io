---
layout:     post   				    
title:      Handler回顾中的思考
subtitle:   Handler机制回顾   #副标题
date:       2020-2-8		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-js-version.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Handler机制回顾系列 
---

# Handler回顾中的思考

## 前言

正常来说这个时间，我应该是在学校开展新一轮的学习以及项目进度的推动，但是由于疫情的波动，一直在家，终于下定决心，在家里等待的同时，把这段时间在家里沉淀以及思考的东西写在笔记里，记录下来

在学习Handler源码的过程中，走到了native层去看到了在Handler中等待唤醒机制的设计，但是并没有对其中两个关键的系统调用进行学习：eventfd、epoll

以及还有就是对Handler中Looper设计的进一步思考，着重点在ThreadLocal如何实现线程独立的角度进行分析

## ThreadLocal

### ThreadLocal-> set

```java
  public void set(T value) {
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null)
           map.set(this, value);
       else
           createMap(t, value);
   }
```

获取当前线程，然后拿到ThreadLocalMap，而ThreadLocal其实是ThreadLocalMap中的key

### ThreadLocal-> getMap

```java
 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
 }
```

可见其实ThreadLocalMap是Thread中的成员变量，而对外我们操作的是ThreadLocal提供的接口，但是ThreadLocal其实只是这个Map的key，所以就可以用这样的方式去保证每个线程只有一个Looper

## eventfd

### Handler中的使用

```java
 mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
```

在Handler中通过eventfd创建了用于实现阻塞唤醒机制的管道

### eventfd

```c
SYSCALL_DEFINE2(eventfd2, unsigned int, count, int, flags)
{
    int fd, error;
    struct file *file;
    // 获取一个没有用的文件描述符
    error = get_unused_fd_flags(flags & EFD_SHARED_FCNTL_FLAGS);
    if (error < 0)
        return error;
    fd = error;
    // 创建一个event文件
    file = eventfd_file_create(count, flags);
    if (IS_ERR(file)) {
        error = PTR_ERR(file);
        goto err_put_unused_fd;
    }
    // 把创建出来的文件和文件描述符进行关联
    fd_install(fd, file);
  
    return fd;

err_put_unused_fd:
    put_unused_fd(fd);

    return error;
}
```

查看eventfd创建eventfd的操作，其实和linux中的sys_open很类似，先找到当前进程中一个无用的文件描述符，然后创建出对应的文件，最后执行文件与文件描述符的关联

而无用文件描述符的寻找以及文件对应文件描述符的关联其实也就是对file_struct中文件描述符表进行处理，所以关键肯定就是eventfd的创建过程

### eventfd_file_create 

```c
static const struct file_operations eventfd_fops = {
#ifdef CONFIG_PROC_FS
    .show_fdinfo    = eventfd_show_fdinfo,
#endif
    .release    = eventfd_release,
    .poll       = eventfd_poll,
    .read       = eventfd_read,
    .write      = eventfd_write,
    .llseek     = noop_llseek,
};


struct file *eventfd_file_create(unsigned int count, int flags)
{
    struct file *file;
    struct eventfd_ctx *ctx;

    if (flags & ~EFD_FLAGS_SET)
        return ERR_PTR(-EINVAL);
    // 通过kmalloc创建一个eventfd_ctx
    ctx = kmalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx)
        return ERR_PTR(-ENOMEM);
    // 赋值操作
    kref_init(&ctx->kref);
    init_waitqueue_head(&ctx->wqh);
    ctx->count = count;
    ctx->flags = flags;
    // 通过anon_inode_getfile获取一个file对象
    file = anon_inode_getfile("[eventfd]", &eventfd_fops, ctx,
                  O_RDWR | (flags & EFD_SHARED_FCNTL_FLAGS));
    if (IS_ERR(file))
        eventfd_free_ctx(ctx);

    return file;
}
```

而对于eventfd来说，每个eventfd中会包含一个eventfd_ctx对象，在这个event_ctx中包含有一个等待队列，这个等待队列的作用是：当进程需要阻塞的时候的时候挂在对应的eventfd的等待队列上，以及还有一个计数器，这个计数器的作用是：一个计数器。这个计数器的作用是当写入数据到当前这个eventfd创建的文件描述符，计数增加，并且唤醒对应的等待队列。当读取数据时候，计数清0，并且返回计数。当然eventfd_signal也会增加计数和唤醒

然后通过anon_inode_getfile去获取一个file对象，然后把eventfd_ctx作为file结构的私有成员变量，并且关联了eventfd自身的操作函数表eventfd_fops

相比于普通文件的创建，eventfd的创建重写了文件描述符对应的read，write，seek，poll，release操作，以及创建了一个event_ctx

回顾在Looper中对阻塞管道的使用，主要对eventfd的操作为read和write，所以我们只需要去阅读eventfd的read和write的实现，就可以知道为什么在Handler机制中要选择这种特殊的文件

### eventfd_read

```c
static ssize_t eventfd_read(struct file *file, char __user *buf, size_t count,
                loff_t *ppos)
{
    // 获取eventfd中的eventfd_ctx
    struct eventfd_ctx *ctx = file->private_data;
    ssize_t res;
    __u64 cnt;
    // 校验读取大小是否满足条件
    if (count < sizeof(cnt))
        return -EINVAL;
    // 调用eventfd_ctx_read，并返回eventfd_ctx中的count计数
    res = eventfd_ctx_read(ctx, file->f_flags & O_NONBLOCK, &cnt);
    if (res < 0)
        return res;
    return put_user(cnt, (__u64 __user *) buf) ? -EFAULT : sizeof(cnt);
}
```

### eventfd_ctx_read

```c
ssize_t eventfd_ctx_read(struct eventfd_ctx *ctx, int no_wait, __u64 *cnt)
{
    ssize_t res;
    DECLARE_WAITQUEUE(wait, current);
    // 加锁保证安全性
    spin_lock_irq(&ctx->wqh.lock);
    *cnt = 0;
    res = -EAGAIN;
    // 如果数据不为0，把res赋值为0
    if (ctx->count > 0)
        res = 0;
    // 如果数据为0，而且允许等待
    else if (!no_wait) {
        // 添加到等待队列中
        __add_wait_queue(&ctx->wqh, &wait);
        // 进行一个死循环，不断检查count的值，如果count大于0，就说明有数据了
        // 然后同样设置res为0，并移除等待队列
        for (;;) {
            set_current_state(TASK_INTERRUPTIBLE);
            if (ctx->count > 0) {
                res = 0;
                break;
            }
            // 中断处理
            if (signal_pending(current)) {
                res = -ERESTARTSYS;
                break;
            }
            spin_unlock_irq(&ctx->wqh.lock);
            schedule();
            spin_lock_irq(&ctx->wqh.lock);
        }
        __remove_wait_queue(&ctx->wqh, &wait);
        __set_current_state(TASK_RUNNING);
    }
    // 调用eventfd_ctx_do_read把值更新到eventfd_ctx中
    if (likely(res == 0)) {
        eventfd_ctx_do_read(ctx, cnt);
        if (waitqueue_active(&ctx->wqh))
            wake_up_locked_poll(&ctx->wqh, POLLOUT);
    }
    spin_unlock_irq(&ctx->wqh.lock);

    return res;
}
```

### eventfd_ctx_do_read

```c
static void eventfd_ctx_do_read(struct eventfd_ctx *ctx, __u64 *cnt)
{
    *cnt = (ctx->flags & EFD_SEMAPHORE) ? 1 : ctx->count;
    ctx->count -= *cnt;
}
```

### eventfd_write

```c
static ssize_t eventfd_write(struct file *file, const char __user *buf, size_t count,
                 loff_t *ppos)
{
    struct eventfd_ctx *ctx = file->private_data;
    ssize_t res;
    __u64 ucnt;
    DECLARE_WAITQUEUE(wait, current);

    if (count < sizeof(ucnt))
        return -EINVAL;
    // 把数据从用户空间拷贝下来
    if (copy_from_user(&ucnt, buf, sizeof(ucnt)))
        return -EFAULT;
    if (ucnt == ULLONG_MAX)
        return -EINVAL;
    // 加锁保证安全性
    spin_lock_irq(&ctx->wqh.lock);
    res = -EAGAIN;
    // 对数据进行长度的判断
    if (ULLONG_MAX - ctx->count > ucnt)
        res = sizeof(ucnt);
    else if (!(file->f_flags & O_NONBLOCK)) {
        __add_wait_queue(&ctx->wqh, &wait);
        for (res = 0;;) {
            set_current_state(TASK_INTERRUPTIBLE);
            if (ULLONG_MAX - ctx->count > ucnt) {
                res = sizeof(ucnt);
                break;
            }
            if (signal_pending(current)) {
                res = -ERESTARTSYS;
                break;
            }
            spin_unlock_irq(&ctx->wqh.lock);
            schedule();
            spin_lock_irq(&ctx->wqh.lock);
        }
        __remove_wait_queue(&ctx->wqh, &wait);
        __set_current_state(TASK_RUNNING);
    }
    if (likely(res > 0)) {
        // 把值写入到event_ctxd的count中
        ctx->count += ucnt;
        if (waitqueue_active(&ctx->wqh))
            wake_up_locked_poll(&ctx->wqh, POLLIN);
    }
    spin_unlock_irq(&ctx->wqh.lock);

    return res;
}
```

eventfd的读写操作与普通文件的读写操作有所区别，区别在于：

* eventfd不会像file的读写一样尝试着构建一个文件在磁盘上，写入缓存后把脏数据写入磁盘。eventfd只会在内存中构建一个名为[eventfd]的虚拟文件，在这个文件中进行通信 

* eventfd 不能像正常的file一样读写大量的数据。其读写是有限制的。所有的写数据都在一个无符号64位的count上，它会记录下所有写进来的数据。但是也正是整个原因累积写入的数据一旦超出这个这个值，将会失败。所以eventfd通常使用来做通知的

* 在eventfd中，无论是读还是写都会先进入一个循环，读数据的时候，如果没有任何读取，将会不断的进行进程调度切换。而数据是写入当前file结构体私有数据中的eventfd_ctx。说明eventfd是支持极其高效率的进程间，进程中的通知通信

eventfd是一个高性能的进程间通信机制，由于数据传输大小的限制，一般是作为进程间，进程内通知来使用，而这其实也很很符合Handler的这种等待唤醒机制的这种应用场景

## epoll

在Handler机制中，它用epoll这个系统调用去实现多路复用I/O，相比于同步非阻塞I/O，多路复用I/O依然会发送阻塞，只不过是多路复用I/O的阻塞是发生select/poll/epoll这样的一些系统用中，而实际上，帮我们去完成轮询工作的不再是我们进程的用户态，而是有了另外的协助者，而相比起阻塞I/O，多路复用I/O虽然在数据准备以及数据拷贝两个阶段都处于阻塞，但是多路复用I/O不需要等到所有的工作完成才进行一个通知，而是其中任意一个工作完成之后就能返回进行可读，这个也是我觉得Handler中选择这种I/O方式的原因

回顾Handler中epoll的调用处

### Looper.cpp::rebuildEpollLocked 

```c++
void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        // 关闭旧的epoll实例
        close(mEpollFd); 
    }
    // 创建新的epoll实例，并注册wake管道
    mEpollFd = epoll_create(EPOLL_SIZE_HINT); 
    struct epoll_event eventItem;
    // 把未使用的数据区域进行置0操作
    memset(& eventItem, 0, sizeof(epoll_event)); 
    // 可读事件
    eventItem.events = EPOLLIN; 
    eventItem.data.fd = mWakeEventFd;
    // 将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        //将request队列的事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
    }
}
```

可以看到在使用epoll去对管道进行注册时，还会对Looper中的Request进行一个注册，而Looper中的这些Request其实是通过Looper的addFd函数添加进来的

所以其实在Handler中除了管道之外还有其他一些通过要监控的文件，所以才会采用多路复用I/O这样一种I/O方式

而既然选择采用了多路复用I/O这样一种I/O方式，那select/poll/epoll中为什么Handler要选择用epoll呢？这就需要做一个对比

### select

```c++
int select (int maxfd, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, struct timeval *timeout);
```

- maxfd：代表要监控的最大文件描述符fd+1
- writefds：监控可写fd
- readfds：监控可读fd
- exceptfds：监控异常fd
- timeout：超时时长

select函数监视的fd分为三类，分别是`writefds`、`readfds`和`exceptfds`，调用后select函数就会阻塞，直到有fd就绪(有数据可读、可写、或者有except)，或者超时，当select函数返回之后，就可以通过遍历fd_set，找到就绪的fd，但是select有一个最大的缺陷就是单个进程对打开的fd是有一定限制的，它由`FD_SETSIZE`限制，默认值是1024，如果要进行修改，就需要重新编译内核

而select和poll的另外一个缺陷就是随着fd数目的增加，可能只是很少一部分监视的fd是活跃的，但是select和poll每次调用都会线性扫描全部的集合，导致性能的损耗

### poll

```c++
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

poll本质上和select没有区别，它将用户传入的数组拷贝到内核控件，然后查询每个fd对应的设备的状态，如果设备就绪就在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，就挂起当前进程，知道设备就绪或者主动超时，被唤醒后它又要再次遍历fd，这个过程经历了多次无无畏的遍历

poll没有最大连接数的限制，原因是因为它是基于链表来储存的，但是存在以下缺陷：

* 大量的fd的数组被整体复制于用户态和内核态之间
* poll还有一个特点是`水平触发`，如果报告了fd后，没有被处理，那么下一次poll时还会再次报告这个fd
* fd增加时，线性扫描导致性能下降

### epoll

不同于select和poll，epoll不再是一个系统调用，而是三个系统调用

```c++
int epoll_create(int size)；
```

用于创建一个epoll句柄，size是指监听的描述符个数，而epoll支持动态扩展，所以`epoll_create`的这个size只代表初次分配的fd的数目

```c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
```

用于对需要监听的文件描述符(fd)执行op操作，比如将fd加入到epoll句柄

```c++
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)
```

用于等待事件的上报 

epoll使用事件的就绪通知方式，通过`epoll_ctl`注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活fd，`epoll_wait`便可以收到通知

epoll的优点：

* 没有最大数目的限制，支持fd上限受操作系统最大文件句柄数影响
* 效率提升，不是轮询的方式，不会随着fd数目的增加效率下降，epoll只会对活跃的fd进行操作，性能不会收到fd总数的限制
* select/poll都需要内核把fd消息通知给用户空间，而epoll是通过内核和用户空间`mmap`同一块共享内存实现的

相比于poll，epoll对fd的操作有两种模式：LT和ET：

* LT：当`epoll_wait`检测到描述符时间发生并将此事件通知应用程序，应用程序可以不立即处理该事件，下次调用`epoll_wait`的时候，会再次响应应用程序并通知该事件
* ET：当`epoll_wait`检测到描述符时间发生并将此事件通知应用程序，应用程序必须立即处理该事件，如果不处理，下次调用`epoll_wait`的时候不会再次响应应用程序并通知该事件

epoll之所以是三个系统调用而不再是一个系统调用，而且相比select和poll有这样一些优点，其中一个原因在于，epoll会在内核中通过一种数据结构长时间维护这些我们要监视的fd，这样就不需要每次都进行拷贝，以及响应时不再需要去遍历集合找到就绪的fd，而在内核中长时间去维护这些需要监视的fd，难免会涉及有频繁的插入，查找和删除的操作发生，所以就需要一种插入，查找和删除效率都不错的数据结构来存放这些文件描述符，红黑树自然就是不二之选了

## 总结

通过对Handler中这些系统调用的学习，其实了解到了Handler使用epoll，eventfd这些系统调用的好处以及细节，作为贯穿整个系统运行的命脉，使用epoll监听大量的事件，并且可以对所有事件的变化快速反应，而对应阻塞唤醒机制的设计，通过eventfd实现在用户态以及内核态进程线程之间的快速通知，数据量不超过一个long型，而且没有真正去把数据写入到磁盘中