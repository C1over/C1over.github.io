---
layout:     post   				    
title:  proguard				 
subtitle:  混淆     #副标题
date:       2018-8-9			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 混淆
---

## Proguard
Proguard被人们熟知的是它的混淆功能，根据Proguard帮助文档的描述，Proguard可以对Java class 文件进行shrink，optimize，obfuscate和preveirfy。obfuscate(混淆)只是其中之一。简要的介绍下这四个功能：
* 压缩（Shrink）: 检测和删除没有使用的类，字段，方法和特性
* 优化（Optimize） : 分析和优化Java字节码
* 混淆（Obfuscate）: 使用简短的无意义的名称，对类，字段和方法进行重命名
* 预检（Preveirfy）: 用来对Java class进行预验证（预验证主要是针对JME开发来说的，Android中没有预验证过程，默认是关闭）<br>
**补充说明：根据proguard-android-optimize.txt对optimize的描述，在Android中使用该功能是有潜在风险的，并不能保证在所有版本的Dalvik虚拟机上正常运行，该选项默认是关闭的，如果开启，请做好全面的测试。在Android项目中，我们在相应module下的build.gradle文件中会看到**<br>
~~~
 buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
~~~
其中 minifyEnabled 为true是开启Proguard的功能，false是关闭。
### Proguard的基本规则
~~~
-keep class cn.hadcn.test.**
-keep class cn.hadcn.test.*
~~~
一颗星表示只是保持该包下的类名，而子包下的类名还是会被混淆；两颗星表示把本包和所含子包下的类名都保持；用以上方法保持类后，你会发现类名虽然未混淆，但里面的具体方法和变量命名还是变了，这时如果既想保持类名，又想保持里面的内容不被混淆，我们就需要以下方法了
~~~
-keep class cn.hadcn.test.* {*;}
~~~
在此基础上，我们也可以使用Java的基本规则来保护特定类不被混淆，比如我们可以用extend，implement等这些Java规则。如下例子就避免所有继承Activity的类被混淆
~~~
-keep public class * extends android.app.Activity
~~~
如果我们要保留一个类中的内部类不被混淆则需要用$符号，如下例子表示保持ScriptFragment内部类JavaScriptInterface中的所有public内容不被混淆。
~~~
-keepclassmembers class cc.ninty.chat.ui.fragment.ScriptFragment$JavaScriptInterface {
   public *;
}
~~~
再者，如果一个类中你不希望保持全部内容不被混淆，而只是希望保护类下的特定内容，就可以使用(以下字段都是xml标签)
init;    //匹配所有构造器<br>
fields;   //匹配所有域<br>
methods;  //匹配所有方法方法<br>
你还可以在fields或methods前面加上private 、public、native等来进一步指定不被混淆的内容，如
~~~
-keep class cn.hadcn.test.One {
    public <methods>;
}
~~~
表示One类下的所有public方法都不会被混淆，当然你还可以加入参数，比如以下表示用JSONObject作为入参的构造函数不会被混淆
~~~
-keep class cn.hadcn.test.One {
   public <init>(org.json.JSONObject);
}
~~~
<br>
有时候你是不是还想着，我不需要保持类名，我只需要把该类下的特定方法保持不被混淆就好，那你就不能用keep方法了，keep方法会保持类名，而需要用keepclassmembers ，如此类名就不会被保持，为了便于对这些规则进行理解，官网给出了以下规则<br>

* 类和类成员，防止被移除或者被重命名-keep，防止被重命名-keepnames

* 仅类成员，防止被移除或者被重命名-keepclassmembers，防止被重命名-keepclassmembernames

* 如果拥有某成员、保留类和类成员，防止被移除或者被重命名-keepclasseswithmembers，防止被重命名-keepclasseswithmembernames

**注意事项**<br>
1）jni方法不可混淆，因为这个方法需要和native方法保持一致

~~~
-keepclasseswithmembernames class * { # 保持native方法不被混淆    
    native <methods>;
}
~~~
2）反射用到的类不混淆(否则反射可能出现问题)；<br>
3）AndroidMainfest中的类不混淆，所以四大组件和Application的子类和Framework层下所有的类默认不会进行混淆。自定义的View默认也不会被混淆；所以像网上贴的很多排除自定义View，或四大组件被混淆的规则在Android Studio中是无需加入的；<br>
4）与服务端交互时，使用GSON、fastjson等框架解析服务端数据时，所写的JSON对象类不混淆，否则无法将JSON解析成对应的对象；<br>
5）使用第三方开源库或者引用其他第三方的SDK包时，如果有特别要求，也需要在混淆文件中加入对应的混淆规则；<br>
6）有用到WebView的JS调用也需要保证写的接口方法不混淆，原因和第一条一样；<br>
7） Parcelable的子类和Creator静态成员变量不混淆，否则会产生Android.os.BadParcelableException异常；<br>
~~~
-keep class * implements Android.os.Parcelable { # 保持Parcelable不被混淆           
    public static final Android.os.Parcelable$Creator *;
}
~~~
8）使用enum类型时需要注意避免以下两个方法混淆，因为enum类的特殊性，以下两个方法会被反射调用，见第二条规则。<br>
~~~
 keepclassmembers enum * {  
    public static **[] values();  
    public static ** valueOf(java.lang.String);  
}
~~~
9）发布一款应用除了设minifyEnabled为ture，你也应该设置zipAlignEnabled为true，像Google Play强制要求开发者上传的应用必须是经过zipAlign的，zipAlign可以让安装包中的资源按4字节对齐，这样可以减少应用在运行时的内存消耗。<br>
### Proguard的工作流程
Proguard工作流程是对输入的jars经过shrink->optimize->obfuscate->preveirfy依次处理，而library jars是input jars运行所依赖的包，比如Java运行时的rt.jar，Android运行时android.jar，这些jars在上述处理过程中不会有任何改变，仅是作为输入jars的依赖。<br>
Q:Proguard是怎么知道哪些类，方法，成员变量等是无用的呢?<br>
A:，以此来确定哪些部分未使用到。类似于hotspot虚拟机对可回收对象的判定，从GC Roots出发，进行可达性的判断，不可达的为可回收对象。Entry Points非常重要，Proguard的压缩，优化，混淆功能是以Entry Point作为依据的(预检不需要以此为依据)。<br>
**对于可达性判断的小简述：可达性算法(GC Roots Tracing)：从GC Roots作为起点开始搜索，那么整个连通图中的对象便都是活对象，对于GC Roots无法到达的对象便成了垃圾回收的对象，随时可被GC回收。**<br>
在压缩过程中，Proguard从Entry Points出发，递归检索，删除那些没有使用到的类和类的成员，在接下来的优化过程中，那些非Entry Points的类和方法会被设置成private，static或final，没有使用到的参数会被移除，有些方法可能会被标记为内联的，在混淆过程中，会对非EntryPoint的类和类的成员进行重命名，也就是用其它无意义的名称代替。我们在配置文件中用-keep保留的部分属于Entry Point，所以不会被重命名。
### Proguard配置文件的依据
说起重命名，为什么需要保留一些类和类的成员(方法和变量)不被重命名呢 ? 原因是Proguard对class文件经过一系列处理后，能保证功能上和原来是一样的，但有些情况它却不能良好的处理，比如我们代码中有些功能依赖于它们原来的名字，如反射功能，native调用(函数签名)等，如果换成其它名字，会出现找不到，不对应的情况，可能引起程序崩溃，或者我们的对外提供了一些功能，必须保持原来的名字，才能保证其它依赖这些功能的模块能正确的运行等。
这就是我们为什么要配置-keep选项的原因之一，还有一个原因是我们要用-keep告诉Proguard程序的入口（带有-keep的选项都会作为Entry Point），以此来确定哪些是未被使用的类和类的成员，方法等，并删除它们，因此，我们要针对我们的项目配置对应的选项。当然Proguard不仅提供了-keep选项，还有一些其它配置选项，比如-dontoptimize 对输入的Java class 文件不进行优化处理，-verbose 生成混淆后的映射文件等。