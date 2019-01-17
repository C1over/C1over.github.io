---
layout:     post   				    
title:      startService启动过程			 
subtitle:   framework    #副标题
date:       2019-1-17			   	# 时间
author:     Cc1over				# 作者
header-img: img/WechatIMG1.jpeg	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - android
    - framework
---

## startService启动过程

### ActivityManagerService

ActivityManagerService是Android的Java framework的服务框架最重要的服务之一。对于Andorid的Activity、Service、Broadcast、ContentProvider四大组件的管理，包含其生命周期都是通过ActivityManagerService来完成的，在分析四大组件之前，我们要先对AMS有一定的了解:

ActivityManagerService的UML图如下图所示：

![](https://raw.githubusercontent.com/C1over/C1over.github.io/master/img/AMS-UML%E5%9B%BE.png)

在app中启动一个service，就一行语句搞定

```java
startService()； //或 binderService()
```

流程图如下：

![](https://raw.githubusercontent.com/C1over/C1over.github.io/master/img/%E5%90%AF%E5%8A%A8%E6%9C%8D%E5%8A%A1%E6%B5%81%E7%A8%8Bpng.png)



当app通过startServcie或bindService方法来生成并启动服务的过程，主要是ActivityManagerService来完成的

* ActivityManagerService通过socket的通信方式向Zygote进程请求生成(fork)用于承载服务的进程ActivityThread。此处讲述启动远程服务的过程，即服务运行于单独的进程中，对于运行本地服务则不需要启动服务的过程

* Zygote通过fork的方法，将zygote进程复制生成新的进程，并将ActivityThread相关的资源加载到新进程
* ActivityManagerService向新生成的ActivityThread进程，通过Binder方式发送生成服务的请求
* ActivityThread启动运行服务，这便于服务启动的简易过程，真正流程远比这服务

### startService启动过程分析

之前的[Android中的Context](https://c1over.github.io/2019/01/16/Context/)也讲述了四大组件与ContextImp之间的关系，这里就不再重复，而startService其实是Context中的一个接口，我们就以ContextWapper为入口：

##### [-> ContextWrapper.startService]

```java
public class ContextWrapper extends Context {
    public ComponentName startService(Intent service) {
        return mBase.startService(service); 
    }
}
```

##### [-> ContextImp.startService]

```java
class ContextImpl extends Context {
    @Override
    public ComponentName startService(Intent service) {
        //当system进程调用此方法时输出warn信息，system进程建立调用startServiceAsUser方法
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser); 
    }
```

##### [-> ContextImp.startServiceCommon]

```java
private ComponentName startServiceCommon(Intent service, UserHandle user) {
    try {
        // 检验service，当service为空则throw异常
        validateServiceIntent(service);
        service.prepareToLeaveProcess();
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(getContentResolver()), getOpPackageName(), user.getIdentifier());
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException("Not allowed to start service " +
                    service + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException("Unable to start service " +
                    service  ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
}
```

上面这段代码其实就是做了一些校验工作以及创建一个ActivityManagerNative类然后转交任务

##### [-> ActivityManagerNative.getDefault()]

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

gDefault位Singleton类型对象，此处采用单例模式，mInstance为IActivityManager类的代理对象，即ActivityManagerProxy

#####  [-> gDefault.get()]

```java
public abstract class Singleton<T> {
    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                //首次调用create()来获取AMP对象
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

#####  [-> create()]

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        //获取名为"activity"的服务，服务都注册到ServiceManager来统一管理
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = asInterface(b);
        return am;
    }
};
```

来到这里，其实就是通过Binder跨进程拿到了注册在ServiceManager中名为activity的服务，而因为与系统进程不是同一个进程，所以拿到的是ActivityManagerProxy对象，而回到刚刚的[-> ContextImp.startServiceCommon]中，客户端拿到了ActivityManagerService的代理类后就调用它的startService方法

#####  [-> ActivityManagerProxy.startService]

```java
public ComponentName startService(IApplicationThread caller, Intent service, String resolvedType, String callingPackage, int userId) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    service.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeString(callingPackage);
    data.writeInt(userId);
    mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
    reply.readException();
    ComponentName res = ComponentName.readFromParcel(reply);
    data.recycle();
    reply.recycle();
    return res;
}
```

mRemote.transact方法是binder通信的客户端发起方法，经过binder驱动，最后回到binder服务端ActivityManagerNative的onTransact()方法，而onTransact方法中通过START_SERVICE_TRANSACTION这个code去告知服务端客户端所请求的方法，然后通过data写入数据传输，包括用于身份验证的token以及aplicationThread，然后以reply接受返回的数据

至此，客户端启动服务的流程就走完了，启动服务的过程就在这里由客户端转接到服务端，在进入服务端的流程之前，先用一张流程图总结客户端中的流程：

![](https://raw.githubusercontent.com/C1over/C1over.github.io/master/img/startService%E6%B5%81%E7%A8%8B.png)

[-> ActiviyManagerNative.onTransact]

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
    ...
     case START_SERVICE_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        IBinder b = data.readStrongBinder();
        //生成ApplicationThreadNative的代理对象，即ApplicationThreadProxy对象
        IApplicationThread app = ApplicationThreadNative.asInterface(b);
        Intent service = Intent.CREATOR.createFromParcel(data);
        String resolvedType = data.readString();
        String callingPackage = data.readString();
        int userId = data.readInt();
        ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
        reply.writeNoException();
        ComponentName.writeToParcel(cn, reply);
        return true;
    }
}
```

在这个方法中我们可以看到，服务端拿到了客户端传输过来data数据以及ApplicationThread这个Binder对象，然后在服务端生成了ApplicationThread的代理对象，以此来达到客户端和服务端相互通信的目的

这里顺带总结一波ApplicationThread的UML图：

![](https://raw.githubusercontent.com/C1over/C1over.github.io/master/img/ApplicationThread_UML.png)

与上面的IActivityManager的Binder通信原理一样，ApplicationThreadProxy作为Binder通信的客户端，ApplicationThreadNative作为Binder通信的服务端，其中ApplicationThread继承自ApplicationThreadNative，重写其中的部分方法

而下面的启动任务就由ActivityManagerService转交给它的一个成员ActiveServices来执行启动服务的操作，由于源码比较赘长，我就不在这里一一展示了，有兴趣的朋友可以去[AndroidXRef](http://androidxref.com/)阅读源码，在这里我们主要分析的是ActiveServices完成的任务：

* 检索服务信息

* 对于非前台进程的调整：
  对于非前台进程调用而需要启动的服务，如果已经有其他的后台服务正在启动中，则通过内部维护的延迟队列进行延迟启动

* 判断服务对应的进程是否存在，不存在则创建进程

上面ActiveServices完成的关键工作，而最后肯定就是会通过ApplicationThreadProxy的transact方法返回Service启动结果，我们找到对应的方法并继续分析

##### [-> ApplicationThreadProxy.scheduleCreateService]

```java
public final void scheduleCreateService(IBinder token, ServiceInfo info, CompatibilityInfo compatInfo, int processState) throws RemoteException {
    Parcel data = Parcel.obtain();
    data.writeInterfaceToken(IApplicationThread.descriptor);
    data.writeStrongBinder(token);
    info.writeToParcel(data, 0);
    compatInfo.writeToParcel(data, 0);
    data.writeInt(processState);
    try {
      mRemote.transact(SCHEDULE_CREATE_SERVICE_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
    } catch (TransactionTooLargeException e) {
        throw e;
    }
    data.recycle();
}
```

 从这个方法开始下述的操作又回到了客户端进程

[-> ApplicationThreadNative.onTransact]

```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
    case SCHEDULE_CREATE_SERVICE_TRANSACTION: {
        data.enforceInterface(IApplicationThread.descriptor);
        IBinder token = data.readStrongBinder();
        ServiceInfo info = ServiceInfo.CREATOR.createFromParcel(data);
        CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
        int processState = data.readInt();
        scheduleCreateService(token, info, compatInfo, processState);
        return true;
    }
    ...
}
```

##### [-> ApplicationThread.scheduleCreateService]

```java
public final void scheduleCreateService(IBinder token, ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
    updateProcessState(processState, false);
    CreateServiceData s = new CreateServiceData(); //准备服务创建所需的数据
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;
    sendMessage(H.CREATE_SERVICE, s);
}
```

##### [-> ActivityThread.H.handleMessage]

```java
public void handleMessage(Message msg) {
    switch (msg.what) {
        ...
        case CREATE_SERVICE:
            handleCreateService((CreateServiceData)msg.obj); 
            break;
        case BIND_SERVICE:
            handleBindService((BindServiceData)msg.obj);
            break;
        case UNBIND_SERVICE:
            handleUnbindService((BindServiceData)msg.obj);
            break;
        case SERVICE_ARGS:
            handleServiceArgs((ServiceArgsData)msg.obj);  // serviceStart
            break;
        case STOP_SERVICE:
            handleStopService((IBinder)msg.obj);
            maybeSnapshot();
            break;
        ...
    }
}
```

##### [-> ActivityThread.handleCreateService] 

```java
private void handleCreateService(CreateServiceData data) {
    //当应用处于后台即将进行GC，而此时被调回到活动状态，则跳过本次gc。
    unscheduleGcIdler();
    LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);

    java.lang.ClassLoader cl = packageInfo.getClassLoader();
    //通过反射创建目标服务对象
    Service service = (Service) cl.loadClass(data.info.name).newInstance();
    ...

    try {
        //创建ContextImpl对象
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);
        //创建Application对象
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
        mServices.put(data.token, service);
  ActivityManagerNative.getDefault().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
    } catch (Exception e) {
        ...
    }
}
```

这个方法算是客户端启动服务的最后了，在这里还会进行一次进程通信，主要的目的就是回调AMS中的接口，告诉AMS这个启动的过程有没有超时，如果出现了ANR可以及时监控，然后在AMS执行它下面的一些操作，然后再回调Service的startCommand方法

#### 总结

![](http://gityuan.com/images/android-service/start_service/start_service_processes.jpg)

* 启动进程采用Binder IPC向system_server进程发起startService请求
* system_server进程接收到请求后，向zygote进程发送创建进程的请求
* zygote进程fork出新的子进程Remote Service进程
* Remote Service进程，通过Binder IPC向sytem_server进程发起attachApplication请求
* system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向remote Service进程发送scheduleCreateService请求
* Remote Service进程的binder线程在收到请求后，通过handler向主线程发送CREATE_SERVICE消息
* 主线程在收到Message后，通过发射机制创建目标Service，并回调Service.onCreate()方法







感谢：

* http://gityuan.com/2016/03/06/start-service/