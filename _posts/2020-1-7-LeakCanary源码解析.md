---
layout:     post   				    
title:      LeakCanary源码解析
subtitle:   Android框架学习   #副标题
date:       2020-1-7		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-mma-4.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android框架学习
---

# LeakCanary源码解析

## 前言

因为前段时间做项目维护的时候在使用了一下LeakCanary，也听不少前辈讲过它的原理，不过现在项目转接给了师弟师妹们之后很想自己也走一遍框架流程，站在源码的角度体会一下作者当时的想法和设计

## LeakCanary-> install

```java
public static @NonNull RefWatcher install(@NonNull Application application) {
   return 
        // 创建AndroidRefWatcherBuilder
        refWatcher(application)
        // 设置用于监听内存泄漏分析结果的Service
        .listenerServiceClass(DisplayLeakService.class)
        // 忽略检测的引用
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        // 构建并初始化RefWatcher 
       .buildAndInstall();
  }
```

## LeakCanary-> refWatcher

```java
public static @NonNull AndroidRefWatcherBuilder refWatcher(@NonNull Context context) {
    return new AndroidRefWatcherBuilder(context);
  }
```

## AndroidRefWatcherBuilder-> listenerServiceClass 

```java
public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }
```

方法的主要作用是设置一个用于监听内存泄漏的分析结果的Service，LeakCanay中传入的是DisplayLeakService，它的作用就是，当有分析结果之后，就会发出一个notification，并且通过这个notifcation来启动DisplayLeakActivity用于展示引用关系

## AndroidRefWatcherBuilder-> excludedRefs

```java
public final T excludedRefs(ExcludedRefs excludedRefs) {
    heapDumpBuilder.excludedRefs(excludedRefs);
    return self();
  }
```

用于设置那些忽略的已知的内存泄漏

## AndroidRefWatcherBuilder-> buildAndInstall

```java
public @NonNull RefWatcher buildAndInstall() {
    // ......
    // 创建最终返回的RefWatcher对象
    RefWatcher refWatcher = ();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        // 设置DisplayLeakActivity为enable状态
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      // 启动监听activity的检测  
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
      // 启动监听fragment的检测  
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }

```

主要逻辑就是创建出RefWatcher返回给调用者，而过程就是先执行build，再根据监听的目标调用install方法

## RefWatchBuilder-> build

```java
 public final RefWatcher build() {
    if (isDisabled()) {
      return RefWatcher.DISABLED;
    }
    // 初始化忽略的已知的内存泄漏集合
    if (heapDumpBuilder.excludedRefs == null) {
      heapDumpBuilder.excludedRefs(defaultExcludedRefs());
    }
    // 初始化 heap dump监听器
    HeapDump.Listener heapDumpListener = this.heapDumpListener;
    if (heapDumpListener == null) {
      heapDumpListener = defaultHeapDumpListener();
    }
    // 初始化 DebuggerController
    DebuggerControl debuggerControl = this.debuggerControl;
    if (debuggerControl == null) {
      debuggerControl = defaultDebuggerControl();
    }
    // 初始化 HeapDumper
    HeapDumper heapDumper = this.heapDumper;
    if (heapDumper == null) {
      heapDumper = defaultHeapDumper();
    }
    // 初始化 WatchExcuter
    WatchExecutor watchExecutor = this.watchExecutor;
    if (watchExecutor == null) {
      watchExecutor = defaultWatchExecutor();
    }
    // 初始化 GcTrigger
    GcTrigger gcTrigger = this.gcTrigger;
    if (gcTrigger == null) {
      gcTrigger = defaultGcTrigger();
    }
    
    if (heapDumpBuilder.reachabilityInspectorClasses == null) {
      heapDumpBuilder.reachabilityInspectorClasses(defaultReachabilityInspectorClasses());
    }

    return new RefWatcher(watchExecutor, debuggerControl, gcTrigger, heapDumper, heapDumpListener,
        heapDumpBuilder);
  }
```

## AndroidRefWatcherBuilder-> buildAndInstall

## ActivityRefWatcher-> install

```java
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }
```

## ActivityLifecycleCallbacksAdapter

```java
  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
       }
   };
```

## RefWatcher-> watch

```java
public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }
```

## RefWatcher-> watch

```java
public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    // 生成一个随机的Key
    String key = UUID.randomUUID().toString();
    // 把这个随机的Key添加到retainedKeys中
    retainedKeys.add(key);
    // 关注点
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
    // 执行一个异步任务
    ensureGoneAsync(watchStartNanoTime, reference);
  }
```

这段逻辑相对比较关键，因为这里就是LeakCanary判断Activity内存泄漏的关键

这里会生成一个随机的UUID，并把它保存在retainedKey

然后创建一个KeyedWeakReference这个弱引用，这个对象包含的activity的引用，随机生成的UUID，当然还会为它注册一个ReferenceQueue

## KeyedWeakReference

```java
final class KeyedWeakReference extends WeakReference<Object> {
  public final String key;
  public final String name;

  KeyedWeakReference(Object referent, String key, String name,
      ReferenceQueue<Object> referenceQueue) {
    super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
    this.key = checkNotNull(key, "key");
    this.name = checkNotNull(name, "name");
  }
}
```

在这个KeyedWeakReference中保存那个唯一的UUID，就是用来做表示，方便后面检查的

## RefWatcher-> ensureGoneAsync

```java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

## RefWatcher-> ensureGone

```java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    
    // 先把ReferenceQueue中的弱引用对应的UUID从retainedKeys中移除
    removeWeaklyReachableReferences();

    // ......
    // 如果reference已经被回收就可以直接返回了，判断依据还是对应的UUID是否在retainedKeys值中	
    if (gone(reference)) {
      return DONE;
    }
    // 使用之前初始化的gc触发执行一次gc
    gcTrigger.runGc();
    // gc完之后再一次把ReferenceQueue中的弱引用对应的UUID从retainedKeys中移除
    removeWeaklyReachableReferences();
    // 如果reference还没有被回收，说明内存泄漏
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      // 获取 heapDump
      HeapDump heapDump = heapDumpBuilder
          .heapDumpFile(heapDumpFile)
          .referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();
      // 通知 heapdumpListener 进行分析
      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }
```

## RefWatcher-> removeWeaklyReachableReferences

```java
private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

这里官方的注释说到，WeakReferences会入队在垃圾收集之前，这个确实是，因为就以CMS来说References的处理是在

清除之前的，好像有点扯远，不过这里的逻辑就是把这个Reference从队列中拿出来，然后在retainedKeys中把唯一的UUID清除

之所以要这么做就是因为在添加到ReferenceQueue之前，在native层就会把Reference做解绑，所以它的get函数就会为null，所以才需要一个UUID做唯一标识

## RefWatcher-> gone

```java
private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
}
```

其实就是判断retainedKeys中是否包含对应的UUID

## gcTrigger-> runGc

```java
@Override public void runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime().gc();
      try {
        Thread.sleep(100);
      } catch (InterruptedException e) {
        throw new AssertionError();
      }
      System.runFinalization();
    }
```

其实就是用Runtime.gc建议去进行gc回收，而这里会加个100ms作为缓存

## 小结

到了这里，其实Activity的内存泄漏检测就告一段落了

install中主要包含了一些初始化的操作，而实际的内存泄漏检测逻辑在RefWatcher中

注册applicationCallback在Activity onDestry的时候拿到Activity的引用，侵入非常低

然后在onDestry的时候给Activity绑定一个弱引用，同时用一个唯一的UUID做标识

内存泄漏的检测其实就是判断这个弱引用是否在队列中，为了防止误判，会有两次判断，第一次判断发现没有在队列中就会主动建议gc然后隔100ms再检查一次

然后后面我们的关注点其实是Fragment的内存泄漏怎么做

## FragmentRefWatcher.Helper-> install

```java
public static void install(Context context, RefWatcher refWatcher) {
      List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();

      if (SDK_INT >= O) {
        // android sdk O 版本及其以上中的 fragment
        fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
      }

      try {
        // support 包中的 fragment
        Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
        Constructor<?> constructor =
            fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
        FragmentRefWatcher supportFragmentRefWatcher =
            (FragmentRefWatcher) constructor.newInstance(refWatcher);
        fragmentRefWatchers.add(supportFragmentRefWatcher);
      } catch (Exception ignored) {
      }

      if (fragmentRefWatchers.size() == 0) {
        return;
      }

      Helper helper = new Helper(fragmentRefWatchers);

      Application application = (Application) context.getApplicationContext();
      // 监测 Activity 的生命周期。
      application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
   }
```

主要的逻辑的就是根据fragment的类型创建一个AndroidOFragmentRefWatcher与FragmentRefWatcher，而且把它们保存到fragmentRefWatchers中

然后根据这两个Watcher创建一个Helper，然后依然会注册一个ActivityLifecycleCallbacks

## FragmentRefWatcher.Helper-> ActivityLifecycleCallbacksAdapter 

```java
private final Application.ActivityLifecycleCallbacks activityLifecycleCallbacks =
        new ActivityLifecycleCallbacksAdapter() {
          @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            for (FragmentRefWatcher watcher : fragmentRefWatchers) {
              watcher.watchFragments(activity);
            }
          }
        };
```

这里就是在Activity create的时候，用两种Fragment的watcher去watch当前Activity中Fragment

## AndroidOFragmentRefWatcher-> watchFragments 

```java
 @Override public void watchFragments(Activity activity) {
    FragmentManager fragmentManager = activity.getFragmentManager();
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
  }
```

## FragmentRefWatcher-> watchFragments

```java
@Override public void watchFragments(Activity activity) {
    if (activity instanceof FragmentActivity) {
      FragmentManager supportFragmentManager =
          ((FragmentActivity) activity).getSupportFragmentManager();
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
    }
  }
```

其实着两个FragmentRefWatcher的主要区别就是监听不同类型的Fragment而已，而实际的逻辑都是通过registerFragmentLifecycleCallbacks实现的，所以不管监听Activity还是Fragment，都要拿到Activity的，不过做的操作是不一样的

## FragmentManager.FragmentLifecycleCallbacks

```java
private final FragmentManager.FragmentLifecycleCallbacks fragmentLifecycleCallbacks =
      new FragmentManager.FragmentLifecycleCallbacks() {

        @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
          View view = fragment.getView();
          if (view != null) {
            refWatcher.watch(view);
          }
        }

        @Override
        public void onFragmentDestroyed(FragmentManager fm, Fragment fragment) {
          refWatcher.watch(fragment);
        }
      };
```

区别于Activity，Fragment 的生命周期的监听是通过 FragmentManager 来注册实现的

而且它监听onFragmentViewDestroyed和onFragmentDestroyed两个方法，这也和Fragment的生命周期有关

而实际的检测方式其实和Activity一样，都是弱引用加引用队列

## 存在问题

虚拟机并没有提供强制触发 GC 的 API ，通过System.gc()或 Runtime.getRuntime().gc()只能建议系统进行 GC ，如果系统忽略了我们的 GC 请求，可回收的对象就不会被加入 ReferenceQueue

将可回收对象加入 ReferenceQueue 需要等待一段时间，LeakCanary 采用延时 100ms 的做法加以规避，但似乎并不绝对管用

监测逻辑是异步的，如果判断 Activity 是否可回收时某个 Activity 正好还被某个方法的局部变量持有，就会引起误判

若反复进入泄漏的 Activity ，LeakCanary 会重复提示该 Activity 已泄漏







参考资料：

[LeakCanary详解与源码分析](<https://juejin.im/post/5c13212af265da61715e37b0>)

[LeakCanary 与 鹅场Matrix ResourceCanary对比分析](<https://www.cnblogs.com/sihaixuan/p/11140479.html>)