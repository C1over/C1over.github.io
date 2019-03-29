---
layout:     post   				    
title:    一个 view的宽高BUG  				 
subtitle:  BUG记录     #副标题
date:       2018-7-17			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - view
---


### 为什么getHeight和getWidth的值会为零
measure过程完成了之后，可以通过getWidth和getHeight去正确获取到View的测量宽/高，但是在一些情况下onMeasure方法中拿到的测量宽高很可能是不准确的**(PS:所以一些好的做法则是在onLayout方法中去获取View的测量宽高或者最终宽高)**<br><br>

而如果在onCreate，onResume，onStart方法中去获取View的宽高均为零，原因则是measure过程和Activity的生命周期是不同步的，因此如果没有测量完，那么这个宽高的值就会为零。<br><br>

### 关于这个问题的几个解决方案
#### 1）Activity/View#onWindowFocusChanged
这个方法的含义：view已经初始化完毕了，宽高都已经准备好了，这个时候去获取宽高信息就变得没有问题了，但是这个方法有一个不好的地方那就是当Activity的窗口得到焦点和失去焦点的时候都会去调用，所以如果频繁地调用onResume和onPause，那么这个方法也会被频繁地调用。

~~~
@Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus) {
            int width = view.getMeasureWidth();
            int height = view.getMeasureHeigh();
        } 
~~~


#### 2）view.post(runnable)
通过post将一个runnable投递带消息队列的尾部，然后等待looper调用这个runnable的时候，view也已经初始化好了。

~~~
this.post(new Runnable()
        {
            @Override
            public void run()
            {
               int width = view.getMeasureWidth();
               int height = view.getMeasureHeigh();
            }
        });
~~~


#### 3）ViewTreeObserver
使用ViewTreeObserver的这里其实采用观察者模式，注册一个观察者来监听view树，当view树的布局、视图树的焦点、视图树将要绘制、视图树滚动等发生改变时，ViewTreeObserver都会收到通知，ViewTreeObserver不能被实例化，可以调用View.getViewTreeObserver()来获得。它里面有许多回调：

~~~
public final class ViewTreeObserver {
    // Recursive listeners use CopyOnWriteArrayList
    private CopyOnWriteArrayList<OnWindowFocusChangeListener> mOnWindowFocusListeners;
    private CopyOnWriteArrayList<OnWindowAttachListener> mOnWindowAttachListeners;
    private CopyOnWriteArrayList<OnGlobalFocusChangeListener> mOnGlobalFocusListeners;
    private CopyOnWriteArrayList<OnTouchModeChangeListener> mOnTouchModeChangeListeners;
    private CopyOnWriteArrayList<OnEnterAnimationCompleteListener> mOnEnterAnimationCompleteListeners;

    // Non-recursive listeners use CopyOnWriteArray
    // Any listener invoked from ViewRootImpl.performTraversals() should not be recursive
    private CopyOnWriteArray<OnGlobalLayoutListener> mOnGlobalLayoutListeners;
    private CopyOnWriteArray<OnComputeInternalInsetsListener> mOnComputeInternalInsetsListeners;
    private CopyOnWriteArray<OnScrollChangedListener> mOnScrollChangedListeners;
    private CopyOnWriteArray<OnPreDrawListener> mOnPreDrawListeners;
    private CopyOnWriteArray<OnWindowShownListener> mOnWindowShownListeners;

    // These listeners cannot be mutated during dispatch
    private ArrayList<OnDrawListener> mOnDrawListeners;
}
~~~


##### OnGlobalLayoutt方法
其中当View树的状态发生改变或者View树的内部View发生了一些可见性的改变的时候，onGlobalLayout方法就会被调用，但是伴随着View树状态的改变，onGlobalLayout会被调用多次
~~~
view.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
         @Override
         public void onGlobalLayout() {
             view.getViewTreeObserver().removeGlobalOnLayoutListenr(this);
               int width = view.getMeasureWidth();
               int height = view.getMeasureHeigh();
          }
 });
~~~


#### 4）view.measure(int widthMeasureSpec,int heightMeasureSpec)
通过手动对View进行measure来得到view的宽高。

情况一：**match_parent：**<br>
没有办法measure出具体的宽高，因为构造这种MeasureSpec需要parentSize，而在这个时候不知道parentSized的<br>
PS：关于MeasureSpec的生成规则<br>
![](https://files.jb51.net/file_images/article/201709/20179592214421.png?20178592252)

情况二：EXACTLY模式

~~~
  int widthMeasureSpec = MeasureSpec.makeMeasureSpec(width's dp,MeasureSpec.EXACTLY);
  int heightMeasureSpec = MeasureSpec.makeMeasureSpec(height's dp,MeasureSpec.EXACTLY);
  view.measure(widthMeasureSpec,heightMeasureSpec);
~~~
情况三：**wrap_content**

利用最大值构建MeasureSpec

~~~
  int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1>>30)-1,MeasureSpec.AT_MOST);
  int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1>>30)-1,MeasureSpec.AT_MOST);
  view.measure(widthMeasureSpec,heightMeasureSpec);
~~~



#### 附加：获取固定宽高
如果要获取的view的width和height是固定的，那么你可以直接使用：

~~~
  View.getMeasureWidth()
  View.getMeasureHeight()
~~~



#### 小结
View的大小由width和height决定。一个View实际上同时有两种width和height值<br>

第一种是measure width和measure height。他们定义了view想要在父View中占用多少width和height（详情见Layout）。measured height和width可以通过getMeasuredWidth() 和 getMeasuredHeight()获得。<br>

第二种是width和height，有时候也叫做drawing width和drawing height。这些值定义了view在屏幕上绘制和Layout完成后的实际大小。这些值有可能跟measure width和height不同。width和height可以通过getWidth()和getHeight获得。 <br>