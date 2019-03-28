---
layout:     post   				    
title:      Android LayoutInflate源码解析			 
subtitle:   LayoutInflater    #副标题
date:       2019-3-28			   	# 时间
author:     Cc1over				# 作者
header-img: img/home-bg-art.jpg	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - android
    - framework
    - view
---

## LayoutInflater

### 引言

由于笔者之前对LayoutInflate.inflate方法的传入参数以及内部实现比较模糊，之前大一考核期看过但是最近在setContentView源码的时候又想不起来，所以决定写一篇笔记去记录一下这个点

### [LayoutInflate->inflate] 

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
                final String name = parser.getName();
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            } catch (XmlPullParserException e) {
                ......
            }
            return result;
        }
    }

```

在上面这段节选代码中，我在这里主要关注的是root和attachToRoot两个方法参数的逻辑，总结如下：

* **如果是Merge标签，则必须依附于一个root，否则抛出异常**

* **如果root为null，attachToRoot将失去作用，设置任何值都没有意义**

* **如果root不为null，attachToRoot设为true，则会给加载的布局文件的指定一个父布局，即root**

* **如果root不为null，attachToRoot设为false，则会将布局文件最外层的所有layout属性进行设置，当该view被添加到父view当中时，这些layout属性会自动生效**

* **在不设置attachToRoot参数的情况下，如果root不为null，attachToRoot参数默认为true**

而对于上面这段代码的逻辑流程，总结如下：

* **寻找布局的根节点，判断布局的合理性**
* **如果是merge标签，则必须依附于一个rootView，否则抛出异常**
* **根据节点名来创建View对象，对应方法：createViewFromTag(root, name, inflaterContext, attrs)**
* **如果设置的Root不为null，则根据当前标签的参数生成LayoutParams**
* **如果不是attachToRoot ，则对这个Tag和创建出来的View设置LayoutParams(ps:此处的params只有当被添加到一个View中的时候才会生效)**
* **inflate子view的tag，对应方法： rInflateChildren(parser, temp, attrs, true)**
* **如果root不为null且是attachToRoot，则添加创建出来的View到root中**

### [LayoutInflate->createViewFromTag]

```java
 View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
            return view;
        } catch (InflateException e) {
            throw e;
        } catch (ClassNotFoundException e) {
            // omit
        } catch (Exception e) {
            // omit
        }
    }
```

上述代码的执行流程如下：

* **处理部分特殊的tag**
* **有mFactory2，则调用mFactory2的onCreateView方法(这个Factory2是供外部自定义实现的接口)**
* **有mFactory，则调用mFactory2的onCreateView方法(这个Factory是供外部自定义实现的接口，和上面的Factory2的区别再去加载view的策略不一致)**
* **有mPrivateFactory，则调用mFactory2的onCreateView方法(这个Factory是供外部自定义实现的接口，可以使实现上面两种策略)**
* **如果三个Factory都没有，则开始自己创建View**
* **如果View的name中不包含 '.' 则说明是系统控件，会在接下来的调用链在name前面加上 'android.view.'**
* **如果name中包含 '.' 则直接调用createView方法**

### [LayoutInflate->createView]

```java
public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }
            Object lastContext = mConstructorArgs[0];
            if (mConstructorArgs[0] == null) {
                // Fill in the context if not already within inflation.
                mConstructorArgs[0] = mContext;
            }
            Object[] args = mConstructorArgs;
            args[1] = attrs;
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            mConstructorArgs[0] = lastContext;
            return view;
        } catch (ClassCastException e) {
        } 
    }
```

上面这个方法的逻辑相对比较简单，简单梳理一下：

* **反射获取这个View的构造器**
* **缓存构造器**
* **使用反射创建 View 对象，这样一个 View 就被创建出来了**

### [ LayoutInflate->rInflateChildren]

至此，根据根节点名来创建View的工作已经完成，因此我们的目光转向inflate子view的tag的流程中

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
```

### [LayoutInlate->rInflate]

```java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
        final int depth = parser.getDepth();
        int type;
        boolean pendingRequestFocus = false;
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
            if (type != XmlPullParser.START_TAG) {
                continue;
            }
            final String name = parser.getName();
            if (TAG_REQUEST_FOCUS.equals(name)) {
                pendingRequestFocus = true;
                consumeChildElements(parser);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {                
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }
        if (pendingRequestFocus) {
            parent.restoreDefaultFocus();
        }
        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

流程如下：

* **首先进行View的合理性校验，include、merge等标签**
* **通过 createViewFromTag 创建出 View 对象**
* **如果是 ViewGroup，则重复以上步骤**
* **add View 到相应的 parent 中**



参考资料：

* https://juejin.im/post/5b4d9c1ae51d4519721b9113