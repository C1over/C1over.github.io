---
layout:     post   				    
title:    Activity 				 
subtitle:  Components        #副标题
date:       2018-7-10			   	# 时间
author:     BY 		Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 四大组件
---


## 1、Acitvity的生命周期
### onCreate（）
* 当Activity第一次创建的时候调用。这个方法里主要是提供给我们做一些初始化操作，如：创建view、绑定数据到view。
### onStart（）
* 该方法的执行表示Activity已经显示了但是还无法和用户交互，只有当执行到onResume方法的时候才可以进行交互
### onResume（）
* 调用到onResume方法后,Activity就可以与用户开始进行交互了，此时Activity就会位于Activity栈的栈顶了。至此一个Activity就完整的呈现在了我们的眼前并可以与之进行交互了。
### onPause（）
* 当系统开始准备停止当前Activity的时候调用，由于当Activity启动另一个Activity时，旧的Activity先onPause然后新的Activity再启动，所以在onPause方法中不宜做过多耗时的工作。在该方法中google给出的建议是存储一些变化的数据同时停止一些类似于动画等消耗CPU的工作。
### onStop（）
* 紧接着onPause方法调用，此时Activity已经不再显示在用户面前了,此时新的Activity可能已经执行到onStart方法或者onResume方法了，所以此时可做一些较为重量级回收操作比方说关于数据库的一些读写操作等。
### onDestroy（）
* 该方法表示Activity生命周期中的最后一个方法，表示Activity方法将会被销毁，此时我们可以做一些回收操作，如：停止未完成的线程和AsyncTask，取消掉handler的message和runnable，关闭Stream，回收Bitmap等等的一些操作来规避内存泄漏
### onRestart（）
* onStop方法之后可能会调用到onRestart方法，这是因为代表的Activity正在被重新启动，然后紧接着就会继续走到onStart和onResume方法中。
### onSaveInstanceState（）/    onRestoreInstanceState（）
* 当Activity是在异常情况下终止的，系统会调用onSaveInstanceState方法来保存当前Activity的状态，这个方法的调用时机是在onStop方法之前，但是和onPause没有既定的时序关系，而当Activity被重新创建后，系统会调用onRestoreInstanceState（调用时机在onStart之前），而且会把Activity销毁时onSaveInstanceState所保存的Bundle对象传递给onRestoreInstanceState和onCreate方法，所以可以通过这两个方法来判断Activity是否被重建。
* **常见的异常状况**   1）系统内存不足   2）旋转屏幕  3）资源相关的系统配置发生改变
## 2、Activity的四种启动方式
###  1）standar模式
* 标准启动模式：该模式下每次启动Activity都会重新创建Activity实例，在这种模式下谁启动了这个Actvitiy，那么这个Activity与被启动的Activity位于启动它的Activity的栈中。
###  2）singleTop模式
* 栈顶复用模式：该模式下如果Activity已经位于栈顶，那么该Activity不会重新创建，同时它的OnNewIntent方法会被调用，通过方法的参数可以取出其中的信息，并且在这种模式如果这个Actvitiy不位于栈顶，那么这个Activity依然会被重新创建。
###  3）singleTask模式
* singleTask指的是一个任务栈中只能存在一个这样的Acitivity。但是需要我们注意的是如果任务栈中没有该Activity的话系统就会帮我们创建一个Acitivity压入栈顶，但是如果存在该Activity的话就会销毁压在该Activity上的所有Activity最终让创建出来的Activity实例处于栈顶，同时也会回掉该Activity的onNewIntent方法。
###  4）singleInstance模式
* 设置了该模式启动的Acitivyt会在一个独立的任务栈中开启，同事该任务栈有且只有一个这样的Activity实例，每次再启动这个Activity的时候就会在该任务栈里重用该Activity同时回掉onNewIntent方法。<br>
* singleInstace与singleTask的区别在于：singleTask启动的Activity在系统层面上来说是可以有多个实例的。比如说应用程序A想调用singleInstance模式下的ActivityA,而应用程序B也同样调用了，那么在应用程序A和B中就会各有一个ActivityA的实例。但如果该ActivityA是singleInstance模式的话，那么无论有多少个应用程序调用它，它都只可能在系统中存在一个实例同时该实例还是位于它自己的一个单独的任务栈中。
### 5）指定启动模式
* addFlags方式指定启动模式优先级高于AndroidManifest方式，两者都存在的时候以**addFlags指定的启动模式为准**
##  3、Activity之间的数据传递
### 1）通过startActivity来进行Activity的传值
* 使用 startActivity(Intent intent)方法来传入一个Intent对象，这个Intent对象可以指明需要跳转的Activity，或者通过Intent对象指定一个action操作

* setAction方式
可以在AndroidManifest.xml中在 <Activity> 元素下指定一个 <intent-filter> 对象，然后其子元素声明一个 <action> 元素，这样我们可以将这个action动作绑定到了这个Activity上。
### 2）通过startActivityForResult方法得到Activity的回传值

~~~
    public void onClick(View v) {
                //得到新打开Activity关闭后返回的数据
                //第二个参数为请求码，可以根据业务需求自己编号
                startActivityForResult(new Intent(MainActivity.this, OtherActivity.class), 1);
            }
        });
    }
    
    /**
     * 为了得到传回的数据，必须在前面的Activity中（指MainActivity类）重写onActivityResult方法
     * 
     * requestCode 请求码，即调用startActivityForResult()传递过去的值
     * resultCode 结果码，结果码用于标识返回数据来自哪个新Activity
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        String result = data.getExtras().getString("result");//得到新Activity 关闭后返回的数据
        Log.i(TAG, result);
    }
~~~

### 3）Appliction全局参数达到值传递的效果
* 和类的静态变量类似，但是这个类作为单独第三个类
### 4）本地存储，然后在下一个Activity中本地获取
* 存在的问题：如果并发读/写，执行读的操作时读出的内容有可能不是最新的内容。


