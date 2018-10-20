---
layout:     post   				    
title:    volley（2）  				 
subtitle:  网络框架     #副标题
date:       2018-8-8			   	# 时间
author:     BY 		Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 网络框架
---


## volley dispatcher
~~~
/**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        // Make sure any currently running dispatchers are stopped.
        stop();  

        //缓存线程工作
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        //默认有4个网络线程工作
        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
~~~
* 在volley中有两个调度器，一个是CacheDispatcher，另一个是NetDispatcher，顾名思义一个是缓存调度器，一个是网络请求调度，忽略掉一些细节后，这篇笔记是关于调度机制的的学习。
#### CacheDispatcher
在volley中CacheDispatcher是一个调度线程
* 成员变量
~~~
 public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");

       //设置优先级，略低于正常线程的优先级        
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        //缓存线程初始化
        mCache.initialize();

        //循环工作
        while (true) {
            try {
                processRequest();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
            }
        }
~~~
run方法里的逻辑很简单，就是设置优先级初始化以及启动工作...接着点进去看看processRequest的实现
~~~
final Request request = (Request) this.mCacheQueue.take();
      request.addMarker("cache-queue-take")
      if(request.isCanceled()) {
        request.finish("cache-discard-canceled");
    } else {
        Entry entry = this.mCache.get(request.getCacheKey());
        if (entry == null) {
            request.addMarker("cache-miss");
            this.mNetworkQueue.put(request);
        } else if (entry.isExpired()) {
            request.addMarker("cache-hit-expired");
            request.setCacheEntry(entry);
            this.mNetworkQueue.put(request);
        } else {
            request.addMarker("cache-hit");
            Response<?> response = request.parseNetworkResponse(new NetworkResponse(entry.data, entry.responseHeaders));
            request.addMarker("cache-hit-parsed");
            if (entry.refreshNeeded()) {
                request.addMarker("cache-hit-refresh-needed");
                request.setCacheEntry(entry);
                response.intermediate = true;
                this.mDelivery.postResponse(request, response, new Runnable() {
                    public void run() {
                        try {
                            CacheDispatcher.this.mNetworkQueue.put(request);
                        } catch (InterruptedException var2) {

                        }
                    }
                });
            } else {
                this.mDelivery.postResponse(request, response);
            }
        }
~~~
这里的逻辑很寻常，从缓存中取出一条请求，如果不存在在这个请求结果的话，则加入到网络调度线程队列中去，如果存在这条缓存结果时，继续往下走，然后判断是否过期了。如果过期了则同样加入到网络线程队列中去，接着判断是否需要刷新，不需要的话则直接返回缓存结果，否则加入到网络缓存队列中去。<br>
目前为止还没有很体现调度机制，所以我们跳出这个processRequest方法，换一种思路去看一下CacheDispatcher中的成员变量。
* WaitingRequestManager
~~~
if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                List<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new ArrayList<Request<?>>();
                }
                request.addMarker("waiting-for-response");
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.d("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
                return true;
            }  else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                request.setNetworkRequestCompleteListener(this);
                if (VolleyLog.DEBUG) {
                    VolleyLog.d("new request, sending to network %s", cacheKey);
                }
                return false;
            }
~~~
首先判断这个集合中是否存在当前请求，如果存在的话则表示重复请求，则需要排队等待。
* ResponseDelivery 
这是一个结果分发器，由上面CacheDispatcher的源码可以知道，在缓存线程进行了一系列的处理之后就会把一个结果调派到结果分发器中去执行。
~~~
public class ExecutorDelivery implements ResponseDelivery {
    private final Executor mResponsePoster;

    public ExecutorDelivery(final Handler handler) {
        this.mResponsePoster = new Executor() {
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

    public ExecutorDelivery(Executor executor) {
        this.mResponsePoster = executor;
    }

    public void postResponse(Request<?> request, Response<?> response) {
        this.postResponse(request, response, (Runnable)null);
    }

    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        this.mResponsePoster.execute(new ExecutorDelivery.ResponseDeliveryRunnable(request, response, runnable));
    }

    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        this.mResponsePoster.execute(new ExecutorDelivery.ResponseDeliveryRunnable(request, response, (Runnable)null));
    }

    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            this.mRequest = request;
            this.mResponse = response;
            this.mRunnable = runnable;
        }

        public void run() {
            if (this.mRequest.isCanceled()) {
                this.mRequest.finish("canceled-at-delivery");
            } else {
                if (this.mResponse.isSuccess()) {
                    this.mRequest.deliverResponse(this.mResponse.result);
                } else {
                    this.mRequest.deliverError(this.mResponse.error);
                }

                if (this.mResponse.intermediate) {
                    this.mRequest.addMarker("intermediate-response");
                } else {
                    this.mRequest.finish("done");
                }

                if (this.mRunnable != null) {
                    this.mRunnable.run();
                }

            }
        }
    }
}
~~~
* 在ExecutorDelivery的实现可以看到，它会和与外界传入的Handle和主线程进行绑定，所以分发消息是在主线程中执行的，这种绑定的设计是比较常见的，就比如picasso里面的调度机制也是由外部传进来一个UIhandler去和主线程绑定，这样做的目的也很显而易见，那就是可以很方便的在调用者的回调中操作UI。
* 在ExecutorDelivery中也是向外公开了三个方法去通知结果分发器任务的执行的结果，响应失败就把错误发送给客户端，响应成功就执行下一步操作。
##### 小结：目前来看CacheDispatch的一些实现已经看完了，然后就抛开一些与请求以及执行任务相关的逻辑，来分析一波调度机制
* 在这里调度其实很简单，就是用一个调度线程，加一个死循环不断的从队列中获取到任务，然后进行相应逻辑的判断，可以根据自己逻辑的需要，在中途把这份请求添加到一个等待队列中排队等待或者其他任务的队列中，如果一系列判断成功之后，就通过结果调度者公开的方法去通知结果调度者执行相应的任务或则发送错误信息。
## NetworkDispatcher
了解了关于缓存线程调度CacheDispatcher的工作原理，它里面会判断缓存是否存在、是否过期以及是否需要刷新等操作，如果不满足的话则需要加入到网络请求队列，从而让NetworkDispatcher去处理它。下面就继续来学习NetworkDispatcher实现过程，看看网络线程调度怎么实现的
* 成员变量
~~~
/** 请求队列 */
    private final BlockingQueue<Request<?>> mQueue;
    /** 执行网络请求的接口 */
    private final Network mNetwork;
    /** 缓存类接口 */
    private final Cache mCache;
    /** 结果分发 */
    private final ResponseDelivery mDelivery;
~~~
* 构建过程
~~~
public NetworkDispatcher(BlockingQueue<Request<?>> queue,
            Network network, Cache cache, ResponseDelivery delivery) {
        mQueue = queue;
        mNetwork = network;
        mCache = cache;
        mDelivery = delivery;
    }
~~~
* run方法实现
~~~
@Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            try {
                processRequest();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
            }
        }
    }
~~~
NetDispatcher的设计与CacheDispatcher很像，也是点击去看看processRequest的实现
~~~
Request request;
        try {
            request = (Request)this.mQueue.take();
            break;
        } catch (InterruptedException var4) {
            if (this.mQuit) {
                return;
            }
        }
    } try {
        request.addMarker("network-queue-take");
        if (request.isCanceled()) {
            request.finish("network-discard-cancelled");
        } else {
            if (Build.VERSION.SDK_INT >= 14) {
                TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
            }
            NetworkResponse networkResponse = this.mNetwork.performRequest(request);
            request.addMarker("network-http-complete");
            if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                request.finish("not-modified");
            } else {
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");
                if (request.shouldCache() && response.cacheEntry != null) {
                    this.mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }
                request.markDelivered();
                this.mDelivery.postResponse(request, response);
            }
        }
    } catch (VolleyError var5) {
        this.parseAndDeliverNetworkError(request, var5);
    } catch (Exception var6) {
        VolleyLog.e(var6,"Unhandled exception %s",new Object[]{var6.toString()});
        this.mDelivery.postError(request,new VolleyError(var6))
    }
~~~
这里的逻辑也很寻常，首先从网络请求队列中取出一条请求，判断它是否取消了，然后解析网络请求结果到response，然后判断这个请求结果是否应该缓存，同时分发这个网络请求结果出去，但是这并没有到结果分发器，而只是把这个网路处理交给Network去执行，这个处理过程就不再展开了，主要就是判断请求的内容是否被修改过，如果没有被修改过，再根据缓存中是否存在该结果，包装返回NetworkResponse。如果已经修改过的话，则获取请求结果，再次包装返回请求结果。然后获得这个Response对象之后经过一些逻辑判断的过程，就会交给结果调度器去执行。
##### NetDispatcher的实现流程也走完，接下来也是抛开一些关于网络的逻辑，分析一波调派实现
这里其实存在两个调派的过程，在tDispatcher中其实也是不断循环从队列中获取请求，然后交给网络任务执行器执行，返回一个结果再调用结果调度器进一步分发任务。
**从上面两个调度过程来，调度线程做的只是轮循取任务，然后做一些判断，然后就把这些任务去做一个分配，添加到其他更合适的队列中或者交给封装好的专门执行任务的类去执行任务，然后最后在调用结果分发器，去分发处理结果后的事件**

