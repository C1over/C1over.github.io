---
layout:     post   				    
title:      ARouter源码解析
subtitle:   Android框架学习   #副标题
date:       2020-1-5		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-mma-4.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android框架学习
---

# ARouter源码解析

## 前言

最近读了很多系统的源码还有前沿技术框架的源码，大脑消耗有点大，趁着周日调整放松一下自己，走一下项目里用的框架的源码，虽然之前就读过，但是总感觉现在再读一次源码会有不一样的收获，最后就是mark下来

## 初始化

### ARouter-> init

```java
public static void init(Application application) {
        if (!hasInit) {
            logger = _ARouter.logger;
            _ARouter.logger.info(Consts.TAG, "ARouter init start.");
            hasInit = _ARouter.init(application);

            if (hasInit) {
                _ARouter.afterInit();
            }

            _ARouter.logger.info(Consts.TAG, "ARouter init over.");
        }
    }
```

根据hasInit变量判断是否已经初始化，然后转调_ARouter的init方法执行初始化

### _ARouter-> init

```java
protected static synchronized boolean init(Application application) {
        mContext = application;
        LogisticsCenter.init(mContext, executor);
        logger.info(Consts.TAG, "ARouter init success!");
        hasInit = true;

        // It's not a good idea.
        // if (Build.VERSION.SDK_INT > Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
        //     application.registerActivityLifecycleCallbacks(new AutowiredLifecycleCallback());
        // }
        return true;
    }
```

继续转调LogisticsCenter的init方法执行初始化

### LogisticsCenter-> init

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;

        try {
            Set<String> routerMap;
            
            // It will rebuild router map every times when debuggable.
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                // These class was generate by arouter-compiler.
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                }
            } else {
                logger.info(TAG, "Load router map from cache.");
                routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
            }
            // ......
            for (String className : routerMap) {
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // This one of root elements, load root.
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // Load interceptorMeta
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // Load providerIndex
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }

            // ......
        } catch (Exception e) {
            throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
        }
    }

```

**方法流程：**

* 如果是处于debug状态下或者versionCode、versionName有更新就会调用**getFileNameByPackageName**方法创建一个routerMap，而由于本地就有缓存，所以versionCode、versionName只不过是用来判定之前没有创建过routerMap的一个标准而已
* 然后就把这个routerMap用sp缓存起来，除了debug或者版本更新这两种情况之外，下次就可以直接从sp中拿到这个routerMap了
* 然后遍历routerMap的内容，由变量名知道是className：
  * 把IRouteRoot加载进Warehouse.groupsIndex中
  * 把IInterceptorGroup加载进Warehouse.interceptorsIndex中
  * 把IProviderGroup加载进Warehouse.providersIndex中

### ClassUtils-> getFileNameByPackageName

```java
public static Set<String> getFileNameByPackageName(Context context, final String packageName) throws PackageManager.NameNotFoundException, IOException, InterruptedException {
        final Set<String> classNames = new HashSet<>();

        List<String> paths = getSourcePaths(context);
        final CountDownLatch parserCtl = new CountDownLatch(paths.size());

        for (final String path : paths) {
            DefaultPoolExecutor.getInstance().execute(new Runnable() {
                @Override
                public void run() {
                    DexFile dexfile = null;

                    try {
                        if (path.endsWith(EXTRACTED_SUFFIX)) {
                            //NOT use new DexFile(path), because it will throw "permission error in /data/dalvik-cache"
                            dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                        } else {
                            dexfile = new DexFile(path);
                        }

                        Enumeration<String> dexEntries = dexfile.entries();
                        while (dexEntries.hasMoreElements()) {
                            String className = dexEntries.nextElement();
                            if (className.startsWith(packageName)) {
                                classNames.add(className);
                            }
                        }
                    } catch (Throwable ignore) {
                        Log.e("ARouter", "Scan map file in dex files made error.", ignore);
                    } finally {
                        if (null != dexfile) {
                            try {
                                dexfile.close();
                            } catch (Throwable ignore) {
                            }
                        }

                        parserCtl.countDown();
                    }
                }
            });
        }

        parserCtl.await();

        Log.d(Consts.TAG, "Filter " + classNames.size() + " classes by packageName <" + packageName + ">");
        return classNames;
    }
```

**方法流程**

* 先通过getSourcePaths拿到应用的所有dex的文件目录
* 然后创建一个CountDownLatch，目的就是为了多线程解析dex，然后以这个CountDownLatch作为结束判定
* 然后遍历这个Dex拿到所有包名是"com.alibaba.android.arouter.routes"的className然后返回，所以在LogisticsCenter的init中，它才能把想要的class根据className反射出来然后添加到Warehouse中

### Warehouse

```java
class Warehouse {
    // Cache route and metas
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    static Map<String, RouteMeta> routes = new HashMap<>();

    // Cache provider
    static Map<Class, IProvider> providers = new HashMap<>();
    static Map<String, RouteMeta> providersIndex = new HashMap<>();

    // Cache interceptor
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    static List<IInterceptor> interceptors = new ArrayList<>();

    static void clear() {
        routes.clear();
        groupsIndex.clear();
        providers.clear();
        providersIndex.clear();
        interceptors.clear();
        interceptorsIndex.clear();
    }
}
```

* 感觉这个Warehouse就是对所有的缓存做了一层封装，然后统一调起来比较方便
* 看路由跳转应该就知道这些缓存是干嘛用的了

## 路由跳转

### ARouter-> getInstance

```java
public static ARouter getInstance() {
        if (!hasInit) {
            throw new InitException("ARouter::Init::Invoke init(context) first!");
        } else {
            if (instance == null) {
                synchronized (ARouter.class) {
                    if (instance == null) {
                        instance = new ARouter();
                    }
                }
            }
            return instance;
        }
    }
```

### ARouter-> build

```java
public Postcard build(String path) {
        return _ARouter.getInstance().build(path);
}
```

### _ARouter-> build

```java
protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return build(path, extractGroup(path));
        }
    }
```

**方法流程：**

* 这里会navigation的方式拿到PathReplaceService，它是IProvider的子接口，然后这里的逻辑其实在拿到path之前进行一步拦截操作，而这个PathReplaceService应该就是一个拦截器的变种，只不过是注解的方式添加的，而不是像OK那样显示添加，具体是不是得看看注解解释的过程怎么做的
* 根据path切出第一个/和第二个/作为group然后转调build方法

### _ARouter-> build

```java
protected Postcard build(String path, String group) {
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return new Postcard(path, group);
        }
    }
```

**方法流程：**

* 在这里有类似的代码拿到一个PathReplaceService对path做处理
* 然后就根据path、group创建一个Postcard返回

### Postcard

```java
public final class Postcard extends RouteMeta {
    // Base
    private Uri uri;
    private Object tag;             // A tag prepare for some thing wrong.
    private Bundle mBundle;         // Data to transform
    private int flags = -1;         // Flags of route
    private int timeout = 300;      // Navigation timeout, TimeUnit.Second
    private IProvider provider;     // It will be set value, if this postcard was provider.
    private boolean greenChannel;
    private SerializationService serializationService;

    // Animation
    private Bundle optionsCompat;    // The transition animation of activity
    private int enterAnim;
    private int exitAnim;
    
    // ......
}
```

### Postcard-> navigation

```java
public Object navigation(Context context, NavigationCallback callback) {
    return ARouter.getInstance().navigation(context, this, -1, callback);
}
```

### ARouter-> navigation

### _ARouter-> navigation

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        try {
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
            // ......
        }

        // ......

        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                /**
                 * Continue process
                 *
                 * @param postcard route meta
                 */
                @Override
                public void onContinue(Postcard postcard) {
                    _navigation(context, postcard, requestCode, callback);
                }

                /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }

                }
            });
        } else {
            return _navigation(context, postcard, requestCode, callback);
        }

        return null;
    }
```

**方法流程：**

* 调用LogisticsCenter.completion处理postcard
* 然后如果设置greenChannel就直接调_navigation，如果没有就可以在这里执行拦截器，执行完成之后再调 _navigation实际跳转

### LogisticsCenter-> completion

```java
public synchronized static void completion(Postcard postcard) {
       
        // ......
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            if (null == groupMeta) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                try {
                    
                    // ......
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                    // ......
                } catch (Exception e) {
                    // .......
                }

                completion(postcard);   // Reload
            }
        } else {
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();

                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }

                    // Save params name which need auto inject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }

                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }

            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must be implememt IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }
```

**方法流程：**

* Postcard只是存储了跳转相关的信息，然后从Warehouse.routes中拿到这个RouteMeta
* 然后如果上一步拿到的RouteMeta为空，可能是为可能只是没加载尽量，这个时候group就发挥作用了，从Warehouse.groupsIndex中拿到IRouteGroup，这次空了就可以直接抛异常了，因为在ARouter-> init的时候已经扫描过指定目录，把它添加到Warehouse中了，然后重调一次completion重新加载就好了
* 如果Postcard对应的RouteMeta能拿得到的话，就把RouteMeta的信息设置到Postcard中去
* 然后对fragment和provider进行特殊处理：
  * provider：如果是个provider，从Warehouse中查找它的类并构造对象，然后将其设置到providers集合以及Postcard中，然后跳过所有拦截器
  * fragment：跳过所有拦截器

但是这个RouteMeta是什么时候生成的呢？？估计应该是在解析注解的时候把，等等看注解解释器的代码确认一下

### _ARouter-> _navigation

```java
private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Navigation in main looper.
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        if (requestCode > 0) {  // Need start for result
                            ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
                        } else {
                            ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
                        }

                        if ((0 != postcard.getEnterAnim() || 0 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
                            ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
                        }

                        if (null != callback) { // Navigation over.
                            callback.onArrival(postcard);
                        }
                    }
                });

                break;
            case PROVIDER:
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }

        return null;
    }
```

**方法流程：**

* 主要的逻辑就是根据Postcard的type做区分处理
* 如果是Activity就构建一个Intent，把Postcard中Bundle传递过去，然后startActivity就可以了
* 如果是Provider就直接返回Postcard的Provider，所以这也解释了为什么在competion要反射到Provider然后设置进去
* 如果是Fragment就创建一个Fragment，然后把Postcard的Bundle传进去返回就可以了

## 总结

到这里其实ARouter的初始化和路由跳转的流程都走完了，剩下的部分其实就是注解解释生成代码了，在此之前先总结一下初始化和路由跳转的流程以及遗留问题

**初始化**

* 初始化中最主要的工作就是遍历apk中dex，然后找到其中指定的目录，然后遍历指定目录下查找指定的一些class，然后调他们的loadInto方法把信息储存到Warehouse中

**路由跳转**

* 根据path做字符串处理拿到group，然后构建出一个Postcard对象
* 然后从Warehouse中的route根据Postcard的path拿到RouteMeta，然后把RouteMeta中的信息转存到Postcard中，然后就可以根据Postcard的type执行不同类型的分发

**疑问**

* 可见其实跳转目的地，type这些信息是存储在Warehouse中route中，接下来的关注点就是注解解释的时候做了什么，估计很可能是初始化指定目录的指定class了

其实答案就是注解解释，在这步解释中会生成RouteRoot、RouteMeta、Provider、ProviderGroup、Interceptor等类，这也是为什么初始化的过程中要把指定目录下的的一些指定的class找出来loadInto到Warehouse中的原因了

