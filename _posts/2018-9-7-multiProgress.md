---
layout:     post   				    
title:  多进程（1）				 
subtitle:  IPC     #副标题
date:       2018-9-7			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - IPC
---


## 关于Android应用多进程
### android:process
* 应用实现多进程需要依赖于android:process这个属性
* 适用元素：Application, Activity, BroadcastReceiver, Service, ContentProvider。
* 通常情况下，这个属性的值应该是”:“开头。表示这个进程是应用私有的。无法在在跨应用之间共用。
* 如果该属性值以小写字母开头，表示这个进程为全局进程。可以被多个应用共用。
#### 应用多进程有什么好处
* 增加APP可用内存：在Android中，默认情况下系统会为每个App分配一定大小的内存。比如从最早的16M到后面的32M或者48M等。具体的内存大小取决于硬件和系统版本。这些有限的内存对于普通的App还算是够用，但是对于展示大量图片的应用来说，显得实在是捉襟见肘。仔细研究一下，你会发现原来系统的这个限制是作用于进程的(毕竟进程是作为资源分配的基本单位)。意思就是说，如果一个应用实现多个进程，那么这个应用可以获得更多的内存。于是，增加App可用内存成了应用多进程的重要原因。
* 独立于主进程：除了增加App可用内存之外，确保使用多进程，可以独立于主进程，确保某些任务的执行和完成。举一个简单的例子，之前的一个项目存在退出的功能，其具体实现为杀掉进程。为了保证某些统计数据上报正常，不受当前进程退出的影响，我们可以使用独立的进程来完成。
#### 多进程的不足与缺点
1） 数据共享问题
* 由于处于不同的进程导致了数据无法共享内容，无论是static变量还是单例模式的实现。
* SharedPreferences 还没有增加对多进程的支持。
* 静态成员和单例模式完全失效
* 线程同步机制失效
* 跨进程共享数据可以通过Intent,Messenger，AIDL等。
2）SQLite容易被锁
* 每个进程可能会使用各自的SQLOpenHelper实例，如果两个进程同时对数据库操作，则会发生SQLiteDatabaseLockedException等异常。
* 解决方法：可以使用ContentProvider来实现或者使用其他存储方式。
3）不必要的初始化
* 多进程之后，每个进程在创建的时候，都会执行自己的Application.onCreate方法。
* 通常情况下，onCreate中包含了我们很多业务相关的初始化，更重要的这其中没有做按照进程按需初始化，即每个进程都会执行全部的初始化。
* 按需初始化需要根据当前进程名称，进行最小需要的业务初始化。
* 需初始化可以选择简单的if else判断，也可以结合工厂模式
#### 关于android:process的其他问题
android:process部分我们提到，如果这个属性值以小写字母开头，那么就是全局的进程，可以被其他应用共用。
 所谓的共用，指的是不同的App的组件运行在同一个指定的进程中。
##### 准备条件
受制于Android系统的安全机制，我们需要做到以下两个准备条件才可以。
* 这个应用使用同样的签名
* 两个应用指定同一个android:sharedUserId的值
### AIDL
#### 一、概述
* AIDL即Android接口定义语言，是用于定义服务器和客户端通信接口的一种描述语言，可以用于生成IPC的代码。从某种意义上说AIDL其实是一个模版，因为在使用过程中，实际起作用的并不是AIDL文件，而是据此而生成的一个IInterface的实例代码，AIDL其实为了避免我们重复编写代码而出现的一个模版。
* 设计AIDL这门语言的目的就是为了实现进程间通信。在Android系统中，每个进程都运行在一块独立的内存中，在其中完成自己的各项活动，与其他进程都分割开来。可是有时候我们又有应用间进行互动的需求，比如传输数据或者任务委托等，AIDL就是为了满足这种需求而诞生的。通过AIDL，可以在一个进程中获取另一个进程的数据和调用其暴露出来的方法，从而满足进程间通信的需求。
* 通常，暴露方法给其他应用进行调用的应用称为服务端，调用其他应用的放的的应用称为客户端，客户端通过绑定服务端的Service来进行交互。
#### 二、语法
AIDL的语法十分简单，与Java语言基本保持一致，需要记住的规则有以下几点：

1）AIDL文件以.aidl为后缀
2）AIDL支持的数据类型分为如下几种：

* 八种基本数据类型：byte、char、short、int、long、float、double、boolean、String、CharSequence
* 实现了Parcelable接口的数据类型
* List 类型。List承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象
* Map类型。Map承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象

3）AIDL文件可以分为两类。一类用来声明实现了Parcelable接口的数据类型，以供其他AIDL文件使用那些非默认支持的数据类型。还有一类是用来定义接口方法，声明要暴露哪些接口给客户端调用，定向Tag就是用来标注这些方法的参数值。
4）定向Tag。定向Tag表示在跨进程通信中数据的流向，用于标注方法的参数值，分为in、out、inout三种。其中in表示数据只能由客户端流向服务端，out表示数据只能由服务端流向客户端，而inout则表示数据可在服务端与客户端之间双向流通。此外，如果AIDL方法接口的参数类型是：基本数据类型、String、CharSequence或者其他AIDL文件定义的方法接口，那么这些参数值得定向Tag默认是且只能是in，所以除了这些这些类型外，其他参数值都需要明确标注使用哪种定向Tag。
5）明确导包。在AIDL文件中需要明确标明引用到的数据类型所在的包名，即使两个文件处于同个包名下
ps:如果不明确导包AS就会出现build-tools\24.0.1\aidl.exe'' finished with non-zero exit value 1的BUG
#### 三、服务端编码
这里来实际完成一个例子作为示范，需要实现的功能是：客户端通过绑定服务端的Service的方式来调用服务端的方法，获取服务端的书籍列表并向其添加书籍，实现应用间的数据共享

首先是服务端的代码
新建一个工程，包名就定义为 com.czy.server
首先，在应用中需要用到一个 Book 类，而 Book 类是两个应用间都需要使用到的，所以也需要在AIDL文件中声明Book类，为了避免出现类名重复导致无法创建文件的错误，这里需要先建立 Book AIDL 文件，之后再创建 Book 类

右键点击新建一个AIDL文件，命名为 Book
![](https://upload-images.jianshu.io/upload_images/2552605-ae9931fc9a51d441.png)
创建完成后，系统就会默认创建一个 aidl 文件夹，文件夹下的目录结构即是工程的包名，Book.aidi 文件就在其中

Book.aidl 文件中会有一个默认方法，可以删除掉
![](https://upload-images.jianshu.io/upload_images/2552605-2b4708f924c996e5.png)
此时就可以来定义Book类了，Book类只包含一个 Name 属性，并使之实现 Parcelable 接口
~~~
public class Book implements Parcelable {

    private String name;

    public Book(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "book name：" + name;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
    }

    public void readFromParcel(Parcel dest) {
        name = dest.readString();
    }

    protected Book(Parcel in) {
        this.name = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
}
~~~
再来修改 Book.aidl 文件，将之改为声明Parcelable数据类型的AIDL文件
~~~
package com.czy.server;

parcelable Book;
~~~
此外，根据一开始的设想，服务端需要暴露给客户端一个获取书籍列表以及一个添加书籍的方法，这两个方法首先要定义在AIDL文件中，命名为 BookController.aidl，注意这里需要明确导包
~~~
package com.czy.server;
import com.czy.server.Book;

interface BookController {

    List<Book> getBookList();

    void addBookInOut(inout Book book);

}
~~~
上面说到，在进程间通信中真正起作用的并不是AIDL文件，而是系统据此而生成的文件，可以在以下目录中查看系统生成的文件，之后需要使用到当中的内部静态抽象类Stub

ps:创建或修改过AIDL文件后需要clean下工程，使系统及时生成我们需要的文件
![](https://upload-images.jianshu.io/upload_images/2552605-76b54cda3a1f4399.png)
然后在来创建一个Service供客户端远程绑定
~~~
public class AIDLService extends Service {

    private static final String TAG = "AIDLService";

    private List<Book> mBooks;

    private BookComponent.Stub mComponentStub;

    @Override
    public void onCreate() {
        super.onCreate();
        mBooks = new ArrayList<>();
        initData();
    }

    public AIDLService() {
        mComponentStub = new BookComponent.Stub() {

            @Override
            public List<Book> getBookList() throws RemoteException {
                return mBooks;
            }

            @Override
            public void addBookInOut(Book book) throws RemoteException {
                if (book != null) {
                    book.setName("Cc1over-NewBook");
                    mBooks.add(book);
                } else {
                    Log.e(TAG, "Book is not be empty");
                }
            }
        };
    }

    private void initData() {
        Book book1 = new Book("Cc1over-Book1");
        Book book2 = new Book("Cc1over-Book2");
        Book book3 = new Book("Cc1over-Book3");
        Book book4 = new Book("Cc1over-Book4");
        Book book5 = new Book("Cc1over-Book5");
        Book book6 = new Book("Cc1over-Book6");
        mBooks.add(book1);
        mBooks.add(book2);
        mBooks.add(book3);
        mBooks.add(book4);
        mBooks.add(book5);
        mBooks.add(book6);
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mComponentStub;
    }
}
~~~
onBind方法返回的就是BookComponent.Stub对象，实现当中定义的两个方法。

最后，服务端还有一个主意事项，所以客户端要能够找到这个Service，可以通过先指定包名，之后再配置Action值或者直接指定Service类名的方式来绑定Service
如果是通过指定Action值的方式来绑定Service，那还需要将Service的声明改为如下所示：
~~~
 <service
            android:name=".AIDLService"
            android:enabled="true"
            android:exported="true"
            tools:ignore="ExportedService">
            <intent-filter>
                <action android:name="com.hebaiyi.cc1overtest.action.action"/>
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </service>
~~~
#### 四、客户端编码
客户端需要在创建一个新的工程，包名命名为com.hebaiyi.cc1overclient

首先，需要把服务端的AIDL文件以及Book类复制过来，将 aidl 文件夹整个复制到和Java文件夹同个层级下，不需要改动任何代码
![](https://upload-images.jianshu.io/upload_images/2552605-4f7fca63ba90a48a.png)
之后，需要创建和服务端Book类所在的相同包名来存放 Book类
![](https://upload-images.jianshu.io/upload_images/2552605-f2176c351d104faf.png)
~~~
public class MainActivity extends AppCompatActivity {

    private final String TAG = "Client-Activity";

    private Button mBtnList, mBtnAdd;
    private BookComponent mComponent;
    private boolean isConnected = false;
    private List<Book> mBooks;

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mComponent = BookComponent.Stub.asInterface(service);
            isConnected = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            isConnected = false;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bindService();
        mBtnList = findViewById(R.id.main_btn_list);
        mBtnAdd = findViewById(R.id.main_btn_add);
        initEvent();
    }

    private void bindService(){
        Intent intent = new Intent();
        intent.setPackage("com.hebaiyi.www.cc1overtest");
        intent.setAction("com.hebaiyi.www.cc1overtest.action");
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if(isConnected){
            unbindService(connection);
        }
    }

    private void log(){
        for(Book book:mBooks){
            Log.d(TAG, book.getName());
        }
    }

    private void initEvent(){
        mBtnAdd.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (isConnected) {
                    try {
                        mBooks = mComponent.getBookList();
                        log();
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        mBtnList.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if(isConnected){
                    try {
                        Book book = new Book("Cc1overClient-Book");
                        mComponent.addBookInOut(book);
                        log();
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
    }
}
~~~
到此为止，客户端和服务端之间的通信已经实现了，客户端获取了服务端的数据，也想服务端传送了数据
#### 五、定向Tag
在AIDL部分的最后，稍微再来总结一下三种定向Tag之间的差别。上边使用的是InOut类型，服务器对数据的改变也同步到客户端，因此可以说两者之间是双向流动的。

In类型的表现形式是：数据只能由客户端传向服务端，服务端对数据的修改不会影响到客户端

Out类型的表现形式是：数据只能有服务端传向客户端，即使客户端向方法接口传入了一个对象，该对象中的属性也是为空的，即不包含任何数据，服务端获取该对象后，对该对象的任何操作，就会同步到客户端这边，所以服务端在获取到客户端以Out方式传来的数据时会为空。
#### AIDL的注册和反注册
之前的 IBookManager 中只提供了获取 书籍列表(getBookList) 和 添加书籍(addBook) 的方法，现在我们将其扩展一下，希望能加入“当新的书本被添加之后，能够通知出去”。这是一种 观察者模式：
由于 AIDL 中无法使用普通接口，所以添加新的 IOnNewBookArrived.aidl 接口，用于回调新书的到来：
~~~
// IOnNewBookArrivedListener.aidl
package com.hebaiyi.www.cc1overtest;
import com.hebaiyi.www.cc1overtest.Book;

interface IOnNewBookArrivedListener {

    void onNewBookArrived(Book book);

}
~~~
在之前的BookComponent接口中添加新的注册监听/取消监听的方法
~~~
interface BookComponent {

    List<Book> getBookList();

    void addBookInOut(inout Book book);

    void registerListener(IOnNewBookArrivedListener listener);
    
    void unregisterListener(IOnNewBookArrivedListener listener);

}
~~~
* 服务端
服务端的操作就是添加注册和反注册的实现，然后去开启一个线程去更新信息，然后遍历观察者去通知他们信息更新。
**注意事项：**
1)  在IPC机制中，对象是不能夸进程直接传输的，对象跨进程传输的实质都是反序列化的过程，Binder会将传递过来的对象重新转化并生成一个新的对象，所以注册和反注册就是不一样的对象就会导致失败。所以在观察者容器中采用RemoteCallbackList<E extends IInterface> 
2) RemoteCallbackList并不是一个List，所以它必须按照下面的方法
~~~
private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new  RemoteCallbackList<IOnNewBookArrivedListener>();
~~~
~~~
 @Override
            public void registerListener(com.hebaiyi.www.cc1overtest.
                                                 IOnNewBookArrivedListener listener)
                    throws RemoteException {
                mListenerList.register(listener);
            }

            @Override
            public void unregisterListener(com.hebaiyi.www.cc1overtest.
                                                   IOnNewBookArrivedListener listener)
                    throws RemoteException {
                mListenerList.unregister(listener);
            }
~~~
~~~
private void onNewBookArrived(Book book) throws RemoteException {
        mBooks.add(book);
        final int N = mListenerList.beginBroadcast();
        for (int i = 0; i < N; i++) {
            IOnNewBookArrivedListener listener = mListeners.getBroadcastItem(i);
            if(listener!=null){
            try{
             listener.onNewBookArrived(book);
            } catch(RemoteException e){
                e.printStackTrace();
            }
         }
     }
  }
~~~
* 客户端
**注意事项：**调用远程服务方法的时候，调用方法会运行在服务端的Binder线程池中，同时客户端线程会被挂起，如果服务端方法执行比较耗时的话就会
~~~
    omission
~~~
#### AIDL原理
在generated目录下，根据我们写的aidl文件BookComponent.aidl，系统会为我们生成一个BookComponent.java这个类，它继承了Iinterface这个接口（ps：所有可以在Binder中传输的接口都需要继承IInterface接口，接下来分析里面的关键方法以及关键的内部类以及属性。
* DESCRIPTOR
~~~
public static abstract class Stub extends android.os.Binder
            implements com.hebaiyi.www.cc1overtest.BookComponent {
        private static final java.lang.String DESCRIPTOR = "com.hebaiyi.www.cc1overtest.BookComponent";

~~~
这是静态内部类Stub中的一个静态常量，可见Stub这个类就是继承自Binder类，而DESCRIPTOR则是Binder的唯一标识，一般用当前Binder的类型来标识。
* asInterface（android.os.IBinder obj）
~~~
 public static com.hebaiyi.www.cc1overtest.BookComponent asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.hebaiyi.www.cc1overtest.BookComponent))) {
                return ((com.hebaiyi.www.cc1overtest.BookComponent) iin);
            }
            return new com.hebaiyi.www.cc1overtest.BookComponent.Stub.Proxy(obj);
        }
~~~
这个方法用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，而这种转换是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象。
* asBinder
这个方法用于返回当前Binder对象
* onTransact
~~~
 @Override
        public boolean onTransact(int code,
                                  android.os.Parcel data,
                                  android.os.Parcel reply,
                                  int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.hebaiyi.www.cc1overtest.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBookInOut: {
                    data.enforceInterface(DESCRIPTOR);
                    com.hebaiyi.www.cc1overtest.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.hebaiyi.www.cc1overtest.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBookInOut(_arg0);
                    reply.writeNoException();
                    if ((_arg0 != null)) {
                        reply.writeInt(1);
                        _arg0.writeToParcel(reply,
                                android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
~~~
这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求的时候，远程请求会通过系统底层封装后交由此方法处理。由上面的源码可以看到，服务端根据code参数去判断客户端所请求的目标方法是什么，然后从参数data中取出目标方法所需的参数（如果目标方法有参数），然后便执行目标方法，当目标方法执行完成之后就想reply中写入返回值（如何目标方法有返回值），onTransact方法执行过程就是这样。注意事项是：如果该方法返回false，那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，详情如下：
首先在AndroidMenifest中声明所需权限
~~~
<permission     android:name="com.example.test1.permission.ACCESS_BOOK_SERVICE"
android:protectionLevel="normal" />
<uses-permission android:name="com.example.test1.permission.ACCESS_BOOK_SERVICE" />
~~~
然后在onTransact方法中进行权限验证
~~~
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws RemoteException {
            // 权限验证
            int check = checkCallingOrSelfPermission("com.example.test1.permission.ACCESS_BOOK_SERVICE");
            Log.d("check:"+check);
            if(check==PackageManager.PERMISSION_DENIED){
                Log.d("Binder 权限验证失败");
                return false;
            }
            // 包名验证
            String packageName=null;
            String[] packages = getPackageManager().getPackagesForUid(getCallingUid());
            if(packages!=null && packages.length>0){
                packageName = packages[0];
            }
            if(!packageName.startsWith("com.example")){
                Log.d("包名验证失败");
                return false;

            }
            return super.onTransact(code, data, reply, flags);
        };
~~~
此外还可以在onBind方法中进行权限
~~~
 @Override
    public IBinder onBind(Intent intent) {
        int check = checkCallingOrSelfPermission("com.ryg.chapter_2.permission.ACCESS_BOOK_SERVICE");
        Log.d(TAG, "onbind check=" + check);
        if (check == PackageManager.PERMISSION_DENIED) {
            return null;
        }
        return mBinder;
    }
~~~
* Proxy#getBoookList
~~~
 @Override
            public java.util.List<com.hebaiyi.www.cc1overtest.Book>
             getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.hebaiyi.www.cc1overtest.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.hebaiyi.www.cc1overtest.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
~~~
由于Proxy#getBookList和Proxy#addBook的执行过程都相似所以这里只分析Proxy#getBookList的实现，这个方法运行在客户端，当客户端远程调用这个方法时，它内部首先创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象_reply和返回值对象List；然后把该方法的参数信写入_data中（如果有参数的话）；接着调用transact方法来发起RPC（远程调用）请求，同时当前线程挂起；然后服务器的onTransact方法就会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果了最后返回_reply中的数据。

ps：由于当客户端发起远程请求的时候，当前线程会被挂起直至服务端进程返回数据，所以远程方法是很耗时的，而由于服务端Binder方法是运行在Binder的线程池中，所以不管是否耗时都应该采用同比方法去实现。而AIDL的作用是方便系统为我们生产固定格式的java文件，它的本质是系统为我们提供的一种快速实现Binder的工具。

补充：关于Binder的两个很重要的方法linkToDeath和unlinkToDeath，由于Binder运行在服务端进程，如果服务端进程异常终止，就会导致Binder连接断裂，会导致远程调用失败，更关键的是，如果不知道Binder连接断裂，那客户端的功能也会受到影响。To解决这个问题，Binder中提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath设置了一个死亡代理，当Binder死亡时，我们就会收到通知，这个时候我们就可以重新发起连接请求从而恢复连接，以下是给Binder设置死亡代理的步骤；
*  声明一个DeathRecipient对象
DeathRecipient是一个借口，其内部只有一个binderDied，我们需要实现这个方法，当Binder死亡的时候，系统就会回调binderDied方法，然后我们就可以移出之前绑定的binder代理并重新绑定远程服务；
* 在客户端设置远程服务成功之后，给Binder设置死亡代理
* Binder的isBinderAlive方法也可以判断Binder是否死亡
~~~
  private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            if (mICalculateInterface == null) {
                return;
            }
            //移除之前绑定的代理并重新绑定远程服务
            mICalculateInterface.asBinder().unlinkToDeath(mDeathRecipient, 0);
            mICalculateInterface = null;
            bindService();
        }
    };
~~~
~~~
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            if (mICalculateInterface == null) {
                return;
            }
            //移除之前绑定的代理并重新绑定远程服务
            mICalculateInterface.asBinder().unlinkToDeath(mDeathRecipient, 0);
            mICalculateInterface = null;
            bindService();
        }
    };
~~~
~~~
@Override
    protected void onDestroy() {
        //如果连接持续，并且Binder未死亡
        if (mICalculateInterface != null && mICalculateInterface.asBinder().isBinderAlive()) {
            try {
                Log.d(TAG, "unregister listener : " + mOnNewPersonArrivedListener);
                mICalculateInterface.unregisterListener(mOnNewPersonArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        unbindService(conn);
        super.onDestroy();
    }
~~~
### Binder连接池
Binder连接池是用来管理所有Binder，而服务端只需要管理这个Binder连接池即可，通过这样的方式实现一个service管理多个Binder，为不同的模块返回不同的Binder，以此解决多模块远程连接时service随着AIDL接口快速膨胀的尴尬情况
以下参照《Android 开发艺术探索》一书，给出一个Binder连接池的案例：
* 先提供两个AIDL接口来模拟多个模块都要使用AIDL的情况：
~~~
interface ISpeak {
    void speak();
}
~~~
~~~
interface ICalculate {
    int add(in int num1,in int num2);
}
~~~
接着分别实现这两个接口
~~~
public class Speak extends Stub {
    public Speak() {
        Log.d("cylog","我被实例化了..............");
    }
    @Override
    public void speak() throws RemoteException {
        int pid = android.os.Process.myPid();
        Log.d("cylog","当前进程ID为："+pid+"-----"+"这里收到了客户端的speak请求");
    }
}
~~~
~~~
public class Calculate extends ICalculate.Stub {
    @Override
    public int add(int num1, int num2) throws RemoteException {
        int pid = android.os.Process.myPid();
        Log.d("cylog", "当前进程ID为："+pid+"----"+"这里收到了客户端的Calculate请求");
        return num1+num2;
    }
}
~~~
可以看见，这两个接口的实现类，都是继承了Interface.Stub类，在上面AIDL的代码中也出现了类似的代码，而这里我们把实现了接口的方法从服务端抽离出来了，其实这个实现类依然是运行在服务端的进程中，从而实现了AIDL接口和服务端的解耦合工作，让服务端不再直接参与AIDL接口方法的实现工作，这样当模块多起来之后，service中的代码就不会显得过于臃肿。而服务端就根据Binder连接池与AIDL接口联系，客户端需要什么Binder，就提供信息给Binder连接池，而连接池根据相应信息返回正确的Binder，这样客户端就能执行特定的操作了。可以说，Binder连接池的思路，非常类似设计模式之中的工厂模式。
* 为Binder连接池创建AIDL接口：IBinderPool.aidl:
~~~
interface IBinderPool {
    IBinder queryBinder(int binderCode);  //查找特定Binder的方法
}
~~~
Q:为什么需要这个接口？<br>
A:我们从上面的分析可以知道，service端并不直接提供具体的Binder，那么客户端和服务端连接的时候就应该返回一个IBinderPool对象，让客户端拿到这个IBinderPool的实例，然后由客户端决定应该用哪个Binder。所以服务端的代码很简单，只需要返回IBinderPool对象即可
* 服务端service的代码
~~~
public class BinderPoolService extends Service {

    private Binder mBinderPool = new BinderPool.BinderPoolImpl();   // 1
    private int pid = Process.myPid();
    @Override
    public IBinder onBind(Intent intent) {
        Log.d("cylog", "当前进程ID为："+pid+"----"+"客户端与服务端连接成功，服务端返回BinderPool.BinderPoolImpl 对象");
        return mBinderPool;
    }
}
~~~
* BinderPool的具体实现<br>
~~~
public class BinderPool {
    public static final int BINDER_SPEAK = 0;
    public static final int BINDER_CALCULATE = 1;

    private Context mContext;
    private IBinderPool mBinderPool;
    private static volatile BinderPool sInstance;
    private CountDownLatch mConnectBinderPoolCountDownLatch;

    private BinderPool(Context context) {               // 1
        mContext = context.getApplicationContext();
        connectBinderPoolService();
    }

    public static BinderPool getInstance(Context context) {     // 2
        if (sInstance == null) {
            synchronized (BinderPool.class) {
                if (sInstance == null) {
                    sInstance = new BinderPool(context);
                }
            }
        }
        return sInstance;
    }

    private synchronized void connectBinderPoolService() {      // 3
        mConnectBinderPoolCountDownLatch = new CountDownLatch(1);
        Intent service = new Intent(mContext, BinderPoolService.class);
        mContext.bindService(service, mBinderPoolConnection,
                Context.BIND_AUTO_CREATE);
        try {
            mConnectBinderPoolCountDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public IBinder queryBinder(int binderCode) {          // 4
        IBinder binder = null;
        try {
            if (mBinderPool != null) {
                binder = mBinderPool.queryBinder(binderCode);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return binder;
    }

    private ServiceConnection mBinderPoolConnection = new ServiceConnection() {   // 5

        @Override
        public void onServiceDisconnected(ComponentName name) {
            
        }

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinderPool = IBinderPool.Stub.asInterface(service);
            try {
                mBinderPool.asBinder().linkToDeath(mBinderPoolDeathRecipient, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mConnectBinderPoolCountDownLatch.countDown();
        }
    };

    private IBinder.DeathRecipient mBinderPoolDeathRecipient = new IBinder.DeathRecipient() {    // 6
        @Override
        public void binderDied() {
            mBinderPool.asBinder().unlinkToDeath(mBinderPoolDeathRecipient, 0);
            mBinderPool = null;
            connectBinderPoolService();
        }
    };

    public static class BinderPoolImpl extends IBinderPool.Stub {     // 7

        public BinderPoolImpl() {
            super();
        }

        @Override
        public IBinder queryBinder(int binderCode) throws RemoteException {
            IBinder binder = null;
            switch (binderCode) {
                case BINDER_SPEAK: {
                    binder = new Speak();
                    break;
                }
                case BINDER_CALCULATE: {
                    binder = new Calculate();
                    break;
                }
                default:
                    break;
            }

            return binder;
        }
    }

}
~~~
从大体上看，这个类完成的功能有实现客户端和服务端的连接，同时内有还有一个静态内部类：BinderPoolImpl，继承了IBinderPool.Stub，这个静态内部类是运行了服务端的，然后下面是关键方法的分析（前两个方法就是平常的单例模式，所以从第三个方法开始）：
1) private synchronized void connectBinderPoolService():这个方法主要用于客户端与服务端建立连接，在方法内部出现了CountDownLatch类，这个类是用于线程同步的，由于bindService()是异步操作，所以如果要确保客户端在执行其他操作之前已经绑定好服务端，就应该先实现线程同步。
2) public IBinder queryBinder(int binderCode):根据具体的binderCode值来获得某个特定的Binder，并返回。
4) private ServiceConnection mBinderPoolConnection = new ServiceConnection(){}：在连接成功的回调中获取客户端和服务端的连接，接着，最后执行了mConnectBinderPoolCountDownLatch.countDown()方法，此时，执行bindService()的线程就会被唤醒。
5) public static class BinderPoolImpl extends IBinderPool.Stub{} :这个类实现了IBinderPool.Stub，内部实现了IBinderPool的接口方法，这个实现类运行在服务端。内部是queryBinder()方法的实现，根据不同的Bindercode值来实例化不同的实现类，比如Speak或者Calculate，并作为Binder返回给客户端。当客户端调用这个方法的时候，实际上已经是进行了一次AIDL方式的跨进程通信。
* 客户端
~~~
public class MainActivity extends Activity {
    private ISpeak mSpeak;
    private ICalculate mCalculate;
    private int pid = android.os.Process.myPid();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                startWork();
            }
        }).start();
    }

    private void startWork() {
        Log.d("cylog","获取BinderPool对象............");
        BinderPool binderPool = BinderPool.getInsance(MainActivity.this);     // 1
        Log.d("cylog","获取speakBinder对象...........");
        IBinder speakBinder = binderPool.queryBinder(BinderPool.BINDER_SPEAK);  // 2
        Log.d("cylog","获取speak的代理对象............");
        mSpeak = (ISpeak) ISpeak.Stub.asInterface(speakBinder);    // 3 
        try {
            mSpeak.speak();     // 4
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d("cylog","获取calculateBinder对象...........");
        IBinder calculateBinder = binderPool.queryBinder(BinderPool.BINDER_CALCULATE);
        Log.d("cylog","获取calculate的代理对象............");
        mCalculate = (ICalculate) ICalculate.Stub.asInterface(calculateBinder);
        try {
            Log.d("cylog",""+mCalculate.add(5,6));
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

}
~~~
<br>
在这里也是手动的实现AIDL帮我们实现的返回代理Binder的工作，而这些操作也是必须的，因为客户端无法直接通过这个打包的Binder和服务端通信，因此客户端必须借助Proxy类来和服务端通信，这里Proxy的作用就是代理的作用，客户端所有的请求全部通过Proxy来代理。
#### 小总结
* （1）为每个业务模块创建AIDL接口，以及实现其接口的方法。
* （2）创建IBinderPool.aidl文件，定义queryBinder(int BinderCode)方法，客户端通过调用该方法返回特定的Binder对象。
* （3）创建BinderPoolService服务端，在onBind方法返回实例化的BinderPool.IBinderPoolImpl对象。
* （4）创建BinderPool类，在该类实现客户端与服务端的连接，解决线程同步的问题，设置Binder的死亡代理等。在onServiceConnected()方法内，获取到IBinderPool的代理对象。此外，IBinderPool的实现类：IBinderPoolImpl是BinderPool的内部类，实现了IBinderPool.aidl的方法：queryBinder()。




本文参考
* https://www.jianshu.com/p/29999c1a93cd
* https://droidyue.com/blog/2017/01/15/android-multiple-processes-summary/
* https://www.jianshu.com/p/6e23037d6d20