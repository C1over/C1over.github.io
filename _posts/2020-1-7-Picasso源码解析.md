---
layout:     post   				    
title:      Picasso源码解析
subtitle:   Android框架学习   #副标题
date:       2020-1-7		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-mma-4.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android框架学习
---

# Picasso源码解析

## Picasso-> get

```java
public static Picasso get() {
    if (singleton == null) {
      synchronized (Picasso.class) {
        if (singleton == null) {
          if (PicassoProvider.context == null) {
            throw new IllegalStateException("context == null");
          }
          singleton = new Builder(PicassoProvider.context).build();
        }
      }
    }
    return singleton;
  }
```

使用单例模式创建Picasso对象，这里的一个细节就是Picasso中是自己注册了一个ContentProvider提供Context的，这样外界就不需要传入一个Context对象

## Picasso.Builder-> build

```java
 public Picasso build() {
      Context context = this.context;

      if (downloader == null) {
        // 默认是采用okhttp做下载器  
        downloader = new OkHttp3Downloader(context);
      }
      if (cache == null) {
        // 配置内存缓存，大小为手机内存的15%  
        cache = new LruCache(context);
      }
      if (service == null) {
        // 线程池，核心池大小为3  
        service = new PicassoExecutorService();
      }
      if (transformer == null) {
        // 请求转换器，默认的请求转换器没有做任何事，直接返回原请求  
        transformer = RequestTransformer.IDENTITY;
      }
      // 统计
      Stats stats = new Stats(cache);
      // 分发器，封装了主线程Handler、线程池，用于任务调度
      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
    }
  }
```

## Picasso

```java
Picasso(Context context, Dispatcher dispatcher, Cache cache, Listener listener,
      RequestTransformer requestTransformer, List<RequestHandler> extraRequestHandlers, Stats stats,
      Bitmap.Config defaultBitmapConfig, boolean indicatorsEnabled, boolean loggingEnabled) {
 
    // ......
    int builtInHandlers = 7; // Adjust this as internal handlers are added or removed.
    int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
    List<RequestHandler> allRequestHandlers = new ArrayList<>(builtInHandlers + extraCount);

    // ResourceRequestHandler needs to be the first in the list to avoid
    // forcing other RequestHandlers to perform null checks on request.uri
    // to cover the (request.resourceId != 0) case.
    allRequestHandlers.add(new ResourceRequestHandler(context));
    if (extraRequestHandlers != null) {
      allRequestHandlers.addAll(extraRequestHandlers);
    }
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);

    this.stats = stats;
    this.targetToAction = new WeakHashMap<>();
    this.targetToDeferredRequestCreator = new WeakHashMap<>();
    this.indicatorsEnabled = indicatorsEnabled;
    this.loggingEnabled = loggingEnabled;
    this.referenceQueue = new ReferenceQueue<>();
    this.cleanupThread = new CleanupThread(referenceQueue, HANDLER);
    this.cleanupThread.start();
  }
```

除了Builder传入参数的初始化之外，Picasso还在在这里初始化它的责任链，用RequestHandler这个集合存储处理不同请求的处理者

## Picasso-> load

```java
 public RequestCreator load(@Nullable Uri uri) {
    return new RequestCreator(this, uri, 0);
  }
```

## RequestCreator

```java
RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    if (picasso.shutdown) {
      throw new IllegalStateException(
          "Picasso instance already shut down. Cannot submit new requests.");
    }
    this.picasso = picasso;
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
  }
```

## Request.Builder

```java
public static final class Builder {
    private Uri uri;
    private int resourceId;
    private String stableKey;
    private int targetWidth;
    private int targetHeight;
    private boolean centerCrop;
    private int centerCropGravity;
    private boolean centerInside;
    private boolean onlyScaleDown;
    private float rotationDegrees;
    private float rotationPivotX;
    private float rotationPivotY;
    private boolean hasRotationPivot;
    private boolean purgeable;
    private List<Transformation> transformations;
    private Bitmap.Config config;
    private Priority priority;
    // ......
}  
```

代码有点多，但是其实Request.Builder中存储的是和图片相关的属性，所以这也为什么，我们要去设置图片相关的信息要等到load之后再去处理

## RequestCreator

```java
public class RequestCreator {
  private static final AtomicInteger nextId = new AtomicInteger();

  private final Picasso picasso;
  private final Request.Builder data;

  private boolean noFade;
  private boolean deferred;
  private boolean setPlaceholder = true;
  private int placeholderResId;
  private int errorResId;
  private int memoryPolicy;
  private int networkPolicy;
  private Drawable placeholderDrawable;
  private Drawable errorDrawable;
  private Object tag;
  // ......
}    
```

之所以来看看RequestCreator的成员变量，其实就是因为有一些平时用的设置不在Request.Builder中，所以其实可以想到应该就是直接设置到RequestCreator中了

而不难发现，其实RequestCreator中的属性主要是和加载有关的信息，而Request.Builder中属性主要是和图片相关的

## RequestCreator-> into

```java
 public void into(ImageView target) {
    into(target, null);
  }
```

## RequestCreator-> into

```java
public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    // 检查是否为主线程，判断依据就是Looper
    checkMain();

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }

    // 如果Request.Builder中uri和ResourcesId为不存在，就调取消请求，并设置占位图
    if (!data.hasImage()) {
      picasso.cancelRequest(target);
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }
   
    // 处理fit这种情况
    if (deferred) {
      // 设置了fit不可以使用resize设置宽高 
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      // 获取ImageView的宽高  
      int width = target.getWidth();
      int height = target.getHeight();
      // 处理宽高为0的情况
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        // 延时处理
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      data.resize(width, height);
    }

    // 创建一个Request
    Request request = createRequest(started);
    // 拿到这个Request的Key
    String requestKey = createKey(request);
    // 判断是否采用内存缓存
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      // 从内存中根据Key取出Bitmap
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
      if (bitmap != null) {
        // 内存中存在就可以取消这个请求，不联网
        picasso.cancelRequest(target);
        // 设置Bitmap到ImageView上
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        // ......
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }
    // 先设置占位图
    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }
    // 把与图片加载相关的信息构建成一个Action交给线程池
    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);
    picasso.enqueueAndSubmit(action);
  }
```

首先一开始调这个方法就先计算一个起始时间，用于后面创建Request，猜测这个时间用于排队

然后根据Looper检测是否在主线程，所以into这个方法只能在主线程调用，不然就会抛异常了

然后再一次判断Request.Builder中的uri和resourcesId是否存在，这是因为如果是构造函数进来的uri没有判空，所以这里再做一次判断

如果采用了fit这种模式，并且拿不到ImageView的宽高，说明ImageView的尺寸还没拿到，所以这个时候就把加载的任务延迟，如果拿到了ImageView的大小，那就直接设置给Request.Builder就好了

然后如果是正常情况的话，那就创建一个Request，然后根据uri和一些图片的信息生成一个Key，作为索引查找内存中是否有这么一个Bitmap，有的话就可以不联网然后把这个Bitmap设置到ImageView上了

如果内存中没有，那就先设置占位图，然后根据一些图片加载的信息，创建一个Action交给提交

## Picasso-> defer

```java
oid defer(ImageView view, DeferredRequestCreator request) {
    // If there is already a deferred request, cancel it.
    if (targetToDeferredRequestCreator.containsKey(view)) {
      cancelExistingRequest(view);
    }
    targetToDeferredRequestCreator.put(view, request);
  }
```

其实这个延时处理就是把图片加载的请求以View为Key缓存到一个Map中，如果这个目标view已经有了一个延时任务，那么就可以把它取消掉了

## RequestCreator-> createRequest

```java
 private Request createRequest(long started) {
    int id = nextId.getAndIncrement();

    Request request = data.build();
    request.id = id;
    request.started = started;

    boolean loggingEnabled = picasso.loggingEnabled;
    if (loggingEnabled) {
      log(OWNER_MAIN, VERB_CREATED, request.plainId(), request.toString());
    }

    Request transformed = picasso.transformRequest(request);
    if (transformed != request) {
      // If the request was changed, copy over the id and timestamp from the original.
      transformed.id = id;
      transformed.started = started;

      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_CHANGED, transformed.logId(), "into " + transformed);
      }
    }

    return transformed;
  }

```

拿到一个自增的id，用AtomicInterger保证线程安全，其实就是用CAS做对数的更新

然后调用Request.Builder的build方法构建一个Request，设置id和起始时间

然后调用了picasso的transformRequest得到一个transformed，然后又把id和起始时间设置给它

## Picasso-> transformRequest

```java
Request transformRequest(Request request) {
    Request transformed = requestTransformer.transformRequest(request);
    if (transformed == null) {
      throw new IllegalStateException("Request transformer "
          + requestTransformer.getClass().getCanonicalName()
          + " returned null for "
          + request);
    }
    return transformed;
  }
```

这里的transformer是一个接口，是由外界实现并把实现类传入到Picasso中，所以这是Picasso对外提供的一种机制，用于让调用者可以在请求构建到请求提交中间修改请求的信息

## RequestCreator-> into

## Picasso-> enqueueAndSubmit

```java
void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (target != null && targetToAction.get(target) != action) {
      // This will also check we are on the main thread.
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    submit(action);
  }
```

在Picasso维护了一个目标View与Action的映射，然后根据目标View从这个映射关系中取出来判断是否为当前要执行的action

如果不是，就取消掉之前设置的请求，并且建立新的关系映射

## Picasso-> submit

```java
void submit(Action action) {
    dispatcher.dispatchSubmit(action);
}
```

## Dispatcher-> dispatchSubmit

```java
void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }
```

submit其实只是其中一个方法，在Dispatcher里面执行任务调度的原理就是采用handler机制

## DispatcherHandler-> handleMessage

```java
@Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;
        }
        case REQUEST_CANCEL: {
          Action action = (Action) msg.obj;
          dispatcher.performCancel(action);
          break;
        }
        case TAG_PAUSE: {
          Object tag = msg.obj;
          dispatcher.performPauseTag(tag);
          break;
        }
        case TAG_RESUME: {
          Object tag = msg.obj;
          dispatcher.performResumeTag(tag);
          break;
        }
        case HUNTER_COMPLETE: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performComplete(hunter);
          break;
        }
        case HUNTER_RETRY: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performRetry(hunter);
          break;
        }
        case HUNTER_DECODE_FAILED: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performError(hunter, false);
          break;
        }
        case HUNTER_DELAY_NEXT_BATCH: {
          dispatcher.performBatchComplete();
          break;
        }
        case NETWORK_STATE_CHANGE: {
          NetworkInfo info = (NetworkInfo) msg.obj;
          dispatcher.performNetworkStateChange(info);
          break;
        }
        case AIRPLANE_MODE_CHANGE: {
          dispatcher.performAirplaneModeChange(msg.arg1 == AIRPLANE_MODE_ON);
          break;
        }
        default:
          Picasso.HANDLER.post(new Runnable() {
            @Override public void run() {
              throw new AssertionError("Unknown handler message received: " + msg.what);
            }
          });
      }
    }
```

## DispatcherHandler-> performSubmit

```java
void performSubmit(Action action) {
    performSubmit(action, true);
  }
```

## DispatcherHandler-> performSubmit

```java
void performSubmit(Action action, boolean dismissFailed) {
    if (pausedTags.contains(action.getTag())) {
      pausedActions.put(action.getTarget(), action);
      // ......
    }

    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

    // ......

    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    hunter.future = service.submit(hunter);
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }

    // ......
  }
```

根据Action的key从hunterMap中拿到对应的BitmapHunter，如果hunter存在就转调hunter的attach方法，绑定一个action

否则就创建一个hunter，然后把这个hunter提交给线程池，并把它添加到缓存中

## BitmapHunter-> forRequest

```java
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();

    // Index-based loop to avoid allocating an iterator.
    //noinspection ForLoopReplaceableByForEach
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }

    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }
```

遍历Picasso中保存的RequestHandler，然后判断RequestHandler能否处理这个请求，如果能处理就直接创建一个BitmapHunter对象绑定对应的RequestHandler返回

如果找不到能处理的RequestHandler，就会给它绑定一个错误的RequestHandler，本质就是抛异常...

## NetworkRequestHandler-> canHandleRequest

```java
@Override public boolean canHandleRequest(Request data) {
    String scheme = data.uri.getScheme();
    return (SCHEME_HTTP.equals(scheme) || SCHEME_HTTPS.equals(scheme));
  }
```

## FileRequestHandler-> canHandleRequest

```java
 @Override public boolean canHandleRequest(Request data) {
    return SCHEME_FILE.equals(data.uri.getScheme());
  }
```

可见，其实从网络和文件这两个RequestHandler的逻辑来看，它们判断能不能处理这个任务，其实就判断uri的开头是不是http、https、file

## BitmapHunter

```java
BitmapHunter(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats, Action action,
      RequestHandler requestHandler) {
    this.sequence = SEQUENCE_GENERATOR.incrementAndGet();
    this.picasso = picasso;
    this.dispatcher = dispatcher;
    this.cache = cache;
    this.stats = stats;
    this.action = action;
    this.key = action.getKey();
    this.data = action.getRequest();
    this.priority = action.getPriority();
    this.memoryPolicy = action.getMemoryPolicy();
    this.networkPolicy = action.getNetworkPolicy();
    this.requestHandler = requestHandler;
    this.retryCount = requestHandler.getRetryCount();
  }
```

主要是对一些关键值的封装，在BitmapHunter中既有和加载相关的策略及缓存，也有调度类的引用，也有封装了加载请求和目标view的action

所以从整个数据传输的角度来看，其实Picasso经过了不止一次的合并和处理：

* 把加载相关的信息储存在RequestCreator中
* 把图片请求相关的信息储存在Request中
* 上述两个数据是又由调用者传递进来的
* 在into这个转折点把两份数据合并成一个Action
* 然后在在Action数据的基础上，把调度类，缓存，RequestHandler等关键引用封装起来就可以交给线程池执行调度了

## DispatcherHandler-> performSubmit

## PicassoExecutorService-> submit

```java
  @Override
  public Future<?> submit(Runnable task) {
    PicassoFutureTask ftask = new PicassoFutureTask((BitmapHunter) task);
    execute(ftask);
    return ftask;
  }
```

把BitmapHunter封了一层，然后转调execute方法，主要目的是要拿到执行结果

## BitmapHunter-> run

```java
@Override public void run() {
    try {
      updateThreadName(data);
        
      // ......
        
      result = hunt();

      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
    } catch (NetworkRequestHandler.ResponseException e) {
      if (!NetworkPolicy.isOfflineOnly(e.networkPolicy) || e.code != 504) {
        exception = e;
      }
      dispatcher.dispatchFailed(this);
    } catch (IOException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (OutOfMemoryError e) {
      StringWriter writer = new StringWriter();
      stats.createSnapshot().dump(new PrintWriter(writer));
      exception = new RuntimeException(writer.toString(), e);
      dispatcher.dispatchFailed(this);
    } catch (Exception e) {
      exception = e;
      dispatcher.dispatchFailed(this);
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }

```

主要逻辑就是hunt方法拿到bitmap，剩下的代码就是根据结果处理任务调度了

## BitmapHunter-> hunt

```java
Bitmap hunt() throws IOException {
    Bitmap bitmap = null;

    if (shouldReadFromMemoryCache(memoryPolicy)) {
      bitmap = cache.get(key);
      if (bitmap != null) {
        // ......
        loadedFrom = MEMORY;
        // ......
        return bitmap;
      }
    }

    networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();
      exifOrientation = result.getExifOrientation();
      bitmap = result.getBitmap();

      // If there was no Bitmap then we need to decode it from the stream.
      if (bitmap == null) {
        Source source = result.getSource();
        try {
          bitmap = decodeStream(source, data);
        } finally {
          try {
            //noinspection ConstantConditions If bitmap is null then source is guranteed non-null.
            source.close();
          } catch (IOException ignored) {
          }
        }
      }
    }

    if (bitmap != null) {
      // ......
      if (data.needsTransformation() || exifOrientation != 0) {
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifOrientation != 0) {
            bitmap = transformResult(data, bitmap, exifOrientation);
            // ......
          }
          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
            // ......
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);
        }
      }
    }

    return bitmap;
  }
```

首先先从内存缓存中尝试拿Bitmap

然后如果内存没有就走RequestHandler的load方法拿Bitmap

然后判断有没有直接拿到Bitmap，没有就从流里decode一下拿出来

然后也是走一下拿到bitmap后的转换过程，给调用者在拿到Bitmap到设置到target之前插入一些自定义操作

## NetworkRequestHandler-> load

```java
 @Override public Result load(Request request, int networkPolicy) throws IOException {
    okhttp3.Request downloaderRequest = createRequest(request, networkPolicy);
    Response response = downloader.load(downloaderRequest);
    ResponseBody body = response.body();

    if (!response.isSuccessful()) {
      body.close();
      throw new ResponseException(response.code(), request.networkPolicy);
    }

    // Cache response is only null when the response comes fully from the network. Both completely
    // cached and conditionally cached responses will have a non-null cache response.
    Picasso.LoadedFrom loadedFrom = response.cacheResponse() == null ? NETWORK : DISK;

    // Sometimes response content length is zero when requests are being replayed. Haven't found
    // root cause to this but retrying the request seems safe to do so.
    if (loadedFrom == DISK && body.contentLength() == 0) {
      body.close();
      throw new ContentLengthException("Received response with 0 content-length header.");
    }
    if (loadedFrom == NETWORK && body.contentLength() > 0) {
      stats.dispatchDownloadFinished(body.contentLength());
    }
    return new Result(body.source(), loadedFrom);
  }
```

主要逻辑就是构建一个downloaderRequest，也就是把Picasso的Request转换成OK的Request的一个过程

然后转调Downloader的load方法加载Bitmap

## NetworkRequestHandler-> createRequest

```java
private static okhttp3.Request createRequest(Request request, int networkPolicy) {
    CacheControl cacheControl = null;
    if (networkPolicy != 0) {
      if (NetworkPolicy.isOfflineOnly(networkPolicy)) {
        cacheControl = CacheControl.FORCE_CACHE;
      } else {
        CacheControl.Builder builder = new CacheControl.Builder();
        if (!NetworkPolicy.shouldReadFromDiskCache(networkPolicy)) {
          builder.noCache();
        }
        if (!NetworkPolicy.shouldWriteToDiskCache(networkPolicy)) {
          builder.noStore();
        }
        cacheControl = builder.build();
      }
    }

    okhttp3.Request.Builder builder = new okhttp3.Request.Builder().url(request.uri.toString());
    if (cacheControl != null) {
      builder.cacheControl(cacheControl);
    }
    return builder.build();
  }
```

这个create方法揭露了一个问题，那就是为什么之前在内存缓存获取和RequestHandler的load方法中间没有本地缓存的读取呢？其实原因就是因为Picasso的本地缓存是依赖OK来做的，所以逻辑就写在NetworkRequestHandler里了

## OkHttp3Downloader-> load  

```java
@NonNull @Override public Response load(@NonNull Request request) throws IOException {
    return client.newCall(request).execute();
  }
```

然后省下的工作只需要交给OK就可以了

## BitmapHunter-> run

## Dispatcher-> dispatchComplete

```java
 void dispatchComplete(BitmapHunter hunter) {
    handler.sendMessage(handler.obtainMessage(HUNTER_COMPLETE, hunter));
  }
```

## DispatcherHandler-> handleMessage

```java
 @Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        
        // ......
        case HUNTER_COMPLETE: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performComplete(hunter);
          break;
        }
        // ......
  }
```

## Dispatcher-> performComplete 

```java
void performComplete(BitmapHunter hunter) {
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
      cache.set(hunter.getKey(), hunter.getResult());
    }
    hunterMap.remove(hunter.getKey());
    batch(hunter);
    if (hunter.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter), "for completion");
    }
  }
```

写入到内存缓存中

从hunterMap中移除

## Dispatcher-> batch

```java
private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {
      return;
    }
    if (hunter.result != null) {
      hunter.result.prepareToDraw();
    }
    batch.add(hunter);
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
  }
```

运用了handler的延时性，发送HUNTER_DELAY_NEXT_BATCH消息，这个消息最后会出发performBatchComplete方法

performBatchComplete里则是通过mainThreadHandler将BitmapHunter的List发送到主线程处理,所以去看看mainThreadHandler的handleMessage方法 

## Dispatcher-> handleMessage

```java
@Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        ...
        case HUNTER_DELAY_NEXT_BATCH: {
          dispatcher.performBatchComplete();
          break;
        }
        ...
      }
}
```

## Dispatcher-> performBatchComplete

```java
void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<>(batch);
    // 清除batch
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }
```

## 总结

把Picasso的流程走了一遍，大概总结一下一些流程：

* get 通过ContentProvider拿到Context，单例创建Picasso对象，Picasso对象的主要作用为管理RequestHandler、Transformer、Cache、targe和Action的映射、线程池等全局信息

* load 主要涉及RequestCreator、Request这两个类，分配用来储存加载相关、图片相关的信息，所以在load之后能对加载和图片进行一些设置

* into 

  * 把load的信息合并封装成Action，然后这个Action回到Picasso对象，因为Picasso对象储存了一些全局信息，要回到Picasso对象中保存targe和Action的映射关系，避免重复或者同一target在同一时间有多个加载任务
  * 然后Picasso使用Dispatcher调度这个Action
  * 然后Dispatcher基于Action的信息再添加上picasso对象中的一些信息(cache、RequestHandler)等组成BitmapHunter交给线程池执行
  * 批量处理，把操作缓存起来，然后通过handler转到主线程执行，避免ANR

  其他

  * fetch，这个方法很特殊，只会把图片加载到本地和内存缓存中，不会直接设置到ImageView，用于给调用者实现预加载







