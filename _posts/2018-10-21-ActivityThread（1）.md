---
layout:     post   				    
title:  ActivityThread				 
subtitle:  framework   #副标题
date:       2018-10-21			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-swift2.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - framework
---

## ActivityThread
ActivityThread代表一个应用进程的主线程（对于应用进程来说，ActivityThread的main函数确实是由该进程的主线程执行），其职责就是调度及执行在该线程中运行的四大组件

#### 关键的方法
1、main()与systemMain()
这个两个方法都是ActivityThread的入口，所执行的主要供作就是实例化一个Activity示例并构建一个Android的运行环境；
main()函数由普通应用进程的主线程调用；
systemMain()函数由系统应用进程调用，如SystemServer(SystemServer希望他内部的service也通过Android运行环境交互，所以其可看成是一个特殊的系统应用进程)
下面来列出这两个方法的关键性代码以及分析一波：
~~~
 @UnsupportedAppUsage
    public static ActivityThread systemMain() {
        // 系统进程在低内存设备上将不会使用硬件加速渲染，因为会增加太多开销
        if (!ActivityManager.isHighEndGfx()) {
            ThreadedRenderer.disable(true);
        } else {
            ThreadedRenderer.enableForegroundTrimming();
        }
        // 实例化ActivityThread对象
        ActivityThread thread = new ActivityThread();
        // 调用自身的attach方法，与APK相关的工作会在attach方法进行
        thread.attach(true, 0);
        return thread;
    }
~~~
~~~
   public static void main（String[] args）{
       // 前面的省略
       // 准备Looper
       Looper.prepareMainLooper();
       // 实例化ActivityThread对象
       ActivityThread thread = new ActivityThread();
       // 调用自身的attach方法，与APK相关的工作会在attach方法进行
       if(sMainThreadHandler == null){
           sMainThreadHandler = thread.getHandler();
       }
       if(false){
           Looper.myLooper.setMessageLogging(new LogPrinter(Log.DEBUG,"ActivityThread"));
       }  
       Looper.loop();
       throw new RuntimeException("Main thread loop unexpectetdly exited");
}
~~~
以上两个方法在实例化ActivityThread对象之后，都调用了thread.attach(boolean b)函数执行接下来的工作，下面分析attach函数
~~~
 @UnsupportedAppUsage
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            // 处理普通应用进程
        }else{
            // 处理系统应用进程
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
UserHandle.myUserId());
         try {
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            } 
            // 注册Configuration变化的回调通知
            ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
                    // 当系统配置发生变化（如语言切换等）时候调用
            }
        }；
        ViewRootImp.addConfigCallback(configChangeCallback);
  }
~~~
attach函数根据参数传递的boolean值来判断要处理的是系统进程还是普通应用进程，同时实例化几个关键变量
Instrumentaion是一个工具类。当它被启用时，系统先创建它，再通过它来创建其他组件。另外，系统和组件之间的交互也将通过Instrumentation来传递，这样，Instrumentation就能监测系统和这些组件的交互情况了。

#### 关键的内部类
private class ApplicationThread extends IApplicationThread.Stub{};
该类是AMS与应用进程进行跨进程交互的重要的类，Android提供了一个IApplicationThread.adil接口，该接口定义了AMS和应用进程之间的交互方法




本文参考：https://www.cnblogs.com/HenryLeeBlog/p/7445006.html