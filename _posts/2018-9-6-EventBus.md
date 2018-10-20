---
layout:     post   				    
title:  EventBus（1）				 
subtitle:  响应式框架     #副标题
date:       2018-9-6			   	# 时间
author:     BY 		Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 响应式框架
---


## EventBus
![](https://upload-images.jianshu.io/upload_images/1797490-88b4a064b9723ef6.png)
EventBus事件主线由四大部分组成：
1) Publisher发布者：用于分发我呢吧的Event事件，在EventBus中通过post方法进行分发传送
2) Subscriber订阅者：用于接收我们的事件，我们在订阅事件中处理我们接收的数据
3) Event事件：任何一个对象都可以作为事件，比如任何字符串，事件是发布者和订阅者之间的通信载体
4) EventBus：类似于中转站，将我们的事件进行对应的分发处理
### EventBus的基本使用
* 添加依赖
~~~
     implementation 'de.greenrobot:eventbus:3.0.0-beta1'
~~~
* 定义一个消息类，该类可以不继承任何基类也不需要实现任何接口
~~~
public class MessageEvent {
 ......
 }
~~~
* 注册
与Android的广播机制类似，这个过程需要在activity中注册evetbus事件，然后定义接收方法
~~~
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    EventBus.getDefault().register(this);

}
@Override
protected void onDestroy() {
    super.onDestroy();
    EventBus.getDefault().unregister(this);
}
~~~
* 产生事件，即发送消息
~~~
EventBus.getDefault().post(messageEvent);
~~~
* 处理消息
在3.0之前，EventBus还没有使用注解方式。消息处理的方法也只能限定于onEvent、onEventMainThread、onEventBackgroundThread和onEventAsync，分别代表四种线程模型。而在3.0之后，消息处理的方法可以随便取名，但是需要添加一个注解@Subscribe，并且要指定线程模型（默认为POSTING），四种线程模型，下面会讲到。
~~~
@Subscribe(threadMode = ThreadMode.POSTING)
public void XXX(MessageEvent messageEvent){
    ... 
}
~~~
### 线程模型
> 在EventBus的事件处理函数中需要指定线程模型，即指定事件处理函数运行所在的想线程。在上面我们已经接触到了EventBus的四种线程模型。那他们有什么区别呢？ 在EventBus中的观察者通常有四种线程模型，分别是PostThread（默认）、MainThread、BackgroundThread与Async。

* 1) POSTING：如果使用事件处理函数指定了线程模型为PostThread，那么该事件在哪个线程发布出来的，事件处理函数就会在这个线程中运行，也就是说发布事件和接收事件在同一个线程。在线程模型为PostThread的事件处理函数中尽量避免执行耗时操作，因为它会阻塞事件的传递，甚至有可能会引起ANR。

* 2) MAIN：如果使用事件处理函数指定了线程模型为MainThread，那么不论事件是在哪个线程中发布出来的，该事件处理函数都会在UI线程中执行。该方法可以用来更新UI，但是不能处理耗时操作。

* 3) BACKGROUND：</b>如果使用事件处理函数指定了线程模型为BackgroundThread，那么如果事件是在UI线程中发布出来的，那么该事件处理函数就会在新的线程中运行，如果事件本来就是子线程中发布出来的，那么该事件处理函数直接在发布事件的线程中执行。在此事件处理函数中禁止进行UI更新操作。

* 4) ASYNC：如果使用事件处理函数指定了线程模型为Async，那么无论事件在哪个线程发布，该事件处理函数都会在新建的子线程中执行。同样，此事件处理函数中禁止进行UI更新操作。

本文参考
* https://segmentfault.com/a/1190000004279679
* https://www.jianshu.com/p/c35f0c545fc9