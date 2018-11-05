---
layout:     post   				    
title:    BrocastReceiver 				 
subtitle:  Components        #副标题
date:       2018-7-10			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 四大组件
---

##  1、BrocastReceiver基本使用
### 1）  自定义广播接收者BroadcastReceiver
* 继承BroadcastReceivre基类
* 必须复写抽象方法onReceive()方法
* **注意事项**：默认情况下，广播接收器运行在 UI 线程，因此，onReceive（）方法不能执行耗时操作，否则将导致ANR。
###  2） 注册广播
####  **静态注册**
* 注册方式：在AndroidManifest.xml里通过<receive>标签声明
属性说明：<br>
~~~
<receiver 
    android:enabled=["true" | "false"]
//此broadcastReceiver能否接收其他App的发出的广播
//默认值是由receiver中有无intent-filter决定的：如果有intent-filter，默认值为true，否则为false
    android:exported=["true" | "false"]
    android:icon="drawable resource"
    android:label="string resource"
//继承BroadcastReceiver子类的类名
    android:name=".mBroadcastReceiver"
//具有相应权限的广播发送者发送的广播才能被此BroadcastReceiver所接收；
    android:permission="string"
//BroadcastReceiver运行所处的进程
//默认为app的进程，可以指定独立的进程
//注：Android四大基本组件都可以通过此属性指定自己的独立进程
    android:process="string" >

//用于指定此广播接收器将接收的广播类型
//本示例中给出的是用于接收网络状态改变时发出的广播
 <intent-filter>
<action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
    </intent-filter>
</receiver>
~~~
#### **动态注册**
* 注册方式：在代码中调用Context.registerReceiver（）方法
示例代码：
~~~
// 选择在Activity生命周期方法中的onResume()中注册
@Override
  protected void onResume(){
      super.onResume();

    // 1. 实例化BroadcastReceiver子类 &  IntentFilter
     mBroadcastReceiver mBroadcastReceiver = new mBroadcastReceiver();
     IntentFilter intentFilter = new IntentFilter();

    // 2. 设置接收广播的类型
    intentFilter.addAction(android.net.conn.CONNECTIVITY_CHANGE);

    // 3. 动态注册：调用Context的registerReceiver（）方法
     registerReceiver(mBroadcastReceiver, intentFilter);
 }
 
// 注册广播后，要在相应位置记得销毁广播
// 即在onPause() 中unregisterReceiver(mBroadcastReceiver)
// 当此Activity实例化时，会动态将MyBroadcastReceiver注册到系统中
// 当此Activity销毁时，动态注册的MyBroadcastReceiver将不再接收到相应的广播。
 @Override
 protected void onPause() {
     super.onPause();
      //销毁在onResume()方法中的广播
     unregisterReceiver(mBroadcastReceiver);
     }
}
~~~
* **注意事项：** 1）因为当优先级更高的应用需要内存的时候，activity在执行完onPause方法以后就会被销毁（即onStop（），onDestroy（）方法不会执行），所以动态注册最好在onResunm（）方法注册，onPause（）方法注销，以免在内存不足的时候会造成内存泄漏。
## 2、BrocastReceiver的使用场景
* 1）App全局监听，这种主要用于在AndroidManifest中静态注册的广播接收器，一般我们在收到该消息后，需要做一些相应的动作，而这些动作与当前App的组件，比如Activity或者Service的是否运行无关，比如我们在集成第三方Push SDK时，一般都会添加一个静态注册的BroadcastReceiver来监听Push消息，当有Push消息过来时，会在后台做一些网络请求或者发送通知等等。
* 2）组件局部监听，这种主要是在Activity或者Service中使用registerReceiver（）动态注册的广播接收器，因为当我们收到一些特定的消息，比如网络连接发生变化时，我们可能需要在当前Activity页面给用户一些UI上的提示，或者将Service中的网络请求任务暂停。所以这种动态注册的广播接收器适合特定组件的特定消息处理。
~~~
  /**
   * 只有当网络改变的时候才会 经过广播。
   */
  @Override
  public void onReceive(Context context, Intent intent) {
  	// TODO Auto-generated method stub
  	//判断wifi是打开还是关闭
  if(WifiManager.WIFI_STATE_CHANGED_ACTION.equals(intent.getAction())){ 
  	//此处无实际作用，只是看开关是否开启
     int wifiState=intent.getIntExtra(WifiManager.EXTRA_WIFI_STATE,0);
       switch (wifiState) {
  		case WifiManager.WIFI_STATE_DISABLED:
  			break;
  		case WifiManager.WIFI_STATE_DISABLING:
  			break;
  		}
  	}
  	//此处是主要代码，
  	//如果是在开启wifi连接和有网络状态下
  if(ConnectivityManager.CONNECTIVITY_ACTION.equals(intent.getAction())){
  		ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
  		NetworkInfo info = intent.getParcelableExtra(ConnectivityManager.EXTRA_NETWORK_INFO);
  		if(NetworkInfo.State.CONNECTED==info.getState()){
  			//连接状态
  			Log.e("pzf", "有网络连接");
  			//执行后续代码
  			//new AutoRegisterAndLogin().execute((String)null);
  			//ps:由于boradCastReciver触发器组件，他和Service服务一样，都是在主线程的，所以，如果你的后续操作是耗时的操作，请new Thread获得AsyncTask等，进行异步操作
  		}else{
  			Log.e("pzf", "无网络连接");
  		}
  	}
~~~
* Manifest清单文件中为广播注册
~~~
<receiver android:name=".NetBroadCastReciver>
   <intent-filter>
         <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
         <action android:name="android.net.wifi.WIFI_STATE_CHANGED" />
         <action android:name="android.net.wifi.STATE_CHANGE" />            </intent-filter>
</receiver>
~~~
##  3、常见的系统广播

| 系统操作                                                     | action                                  |
| ------------------------------------------------------------ | --------------------------------------- |
| 关闭或打开飞行模式                                           | Intent.ACTION_AIRPLANE_MODE_CHANGED     |
| 充电时或电量发生变化                                         | Intent.ACTION_BATTERY_CHANGED           |
| 电池电量低                                                   | Intent.ACTION_BATTERY_LOW               |
| 电池电量充足（即从电量低变化到饱满时会发出广播               | Intent.ACTION_BATTERY_OKAY              |
| 系统启动完成后(仅广播一次)                                   | Intent.ACTION_BOOT_COMPLETED            |
| 检测网络变化                                                 | ConnectivityManager.CONNECTIVITY_ACTION |
| 按下照相时的拍照按键(硬件按键)时                             | Intent.ACTION_CAMERA_BUTTON             |
| 屏幕锁屏                                                     | Intent.ACTION_CLOSE_SYSTEM_DIALOGS      |
| 设备当前设置被改变时(界面语言、设备方向等)                   | Intent.ACTION_CONFIGURATION_CHANGED     |
| 插入耳机时                                                   | Intent.ACTION_HEADSET_PLUG              |
| 未正确移除SD卡但已取出来时(正确移除方法:设置–SD卡和设备内存–卸载SD卡) | Intent.ACTION_MEDIA_BAD_REMOVAL         |
| 插入外部储存装置（如SD卡）                                   | Intent.ACTION_MEDIA_CHECKING            |
| 成功安装APK                                                  | Intent.ACTION_PACKAGE_ADDED             |
| 成功删除APK                                                  | Intent.ACTION_PACKAGE_REMOVED           |
| 重启设备                                                     | Intent.ACTION_REBOOT                    |
| 屏幕被关闭                                                   | Intent.ACTION_SCREEN_OFF                |
| 屏幕被打开                                                   | Intent.ACTION_SCREEN_ON                 |
| 关闭系统时                                                   | Intent.ACTION_SHUTDOWN                  |
| 重启设备                                                     | Intent.ACTION_REBOOT                    |
##  4、Be Careful
#### 对不同注册方式的广播接收者回调OnReceive（Context context，Intent intent）中的context的返回值不一样
* 对静态注册（全局+应用内广播），回调onReceive（context，intent）中的context返回值是：ReceiverRestrictedContext
* 对全局广播的动态注册，回调onReceive（context，intent）中国的context返回值是：Activity Context
* 对于应用内广播的动态注册（LocalBroadcastManager方式），回调onReceive(context, intent)中的context返回值是：Application Context。
对于应用内广播的动态注册（非LocalBroadcastManager方式），回调onReceive(context, intent)中的context返回值是：Activity Context；