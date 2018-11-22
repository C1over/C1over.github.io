---
layout:     post   				    
title:    ContentProvider  				 
subtitle:  components     #副标题
date:       2018-7-10			   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android
    - 四大组件
---


## ContentProvider的基本用法
#### ContentProvider的使用步骤
1）创建数据库、表，因为内容提供者主要是把私有数据库数据提供给外部访问。
2）写一个类继承ContentProvider类，并重写其中的onCreate（）、query（）、insert（）、update（）、delete（）、getType（）等方法。
3）确定主机名(authority)，添加路径匹配规则，此处会利使用到UriMatcher类。
4）在 AndroidMainifest.xml中声明内容提供者，并添加关键属性android:authorities。
5）编写内容提供者的onCreate（）、query（）、insert（）、update（）、delete（）、getType（）等方法。
#### 关于getType（）方法以及MIMEType：
* 根据给定的 URI 来实现处理 MIME类型的请求, 对于单条记录返回的 MIME 类型是以 vnd.android.cursor.item 开始的, 对于多条记录返回的MIME类型是以vnd.android.cursor.dir/ 开始的 这个方法可以在多线程环境下被调用。
~~~
 @Override  
    public String getType(Uri uri) {  
        int flag = URI_MATCHER.match(uri);  
        switch (flag) {  
            case STUDENT:  
                return "vnd.android.cursor.item";  
            case STUDENTS:  
                return "vnd.android.cursor.dir/";  
        }  
        return null;  
    }  
~~~
#### UriMatch
 * UriMatcher 类的概要描述
    1）这是一个在 content provider 中帮助匹配  URIs 的实用类。
    2）public void addURI (String authority, String path, int code)这个方法是用来表示在  content provider 里面添加外部对其的匹配的规则，当 URI 被匹配的时  候，就会返回  code 码， URI节点可以精确的匹配字符串， "*" 号匹配任何的字符串 "#" 号只能匹配数字。[比如删除一条记录中 ID 往往是数字]<br>
 * 参数说明：
    1）authority : 授权, 就是 AndroidMainifest.xml 中的授权<br>
    2）path : 匹配路径(通常是一个表名)[* 可以作为匹配任意字符的通配符, # 可以作为匹配数字的通配符]。【注意这里如果是单条记录，需要添加 /# 标示符】<br>
~~~
 private static final UriMatcher URI_MATCHER = new UriMatcher(UriMatcher.NO_MATCH);  
    private static final int STUDENT = 1; // 操作单条记录  
    private static final int STUDENTS = 2; // 操作多条记录  
    // 添加对外部的匹配规则  
    static {  
        URI_MATCHER.addURI("com.android.contentproviderdemo.StudentProvider",  
                "student", STUDENTS);  
        URI_MATCHER.addURI("com.android.contentproviderdemo.StudentProvider",  
                "student/#", STUDENT);  
    }  
~~~
