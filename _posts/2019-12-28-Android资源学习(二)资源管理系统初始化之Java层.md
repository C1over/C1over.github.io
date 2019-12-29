---
layout:     post   				    
title:      Android资源学习(二)资源管理系统初始化之Java层
subtitle:   Android资源学习系列   #副标题
date:       2019-12-28		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-keybord.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android资源学习系列
---

# Android资源学习(二)资源管理系统初始化

> 本文源码基于Android9.0

## 前言

这篇文章的目的，主要是专注于Android中在Java层中资源管理系统的初始化，在日常开发中，经常会用到**getResources.getXXX**，所以说明Resource肯定在某个时机进行的初始化，然后把aapt编译的资源加载进来，

而getResources的起源就是**ContextImpl**里的mResources，所以源码起源就是**ContextImpl**的初始化了

## # ContextImpl-> createActivityContext

```java
static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId, Configuration overrideConfiguration) {
        // ...
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,activityToken, null, 0, classLoader, null);
        final ResourcesManager resourcesManager = ResourcesManager.getInstance();
        context.setResources(resourcesManager.createBaseActivityResources(activityToken,
                packageInfo.getResDir(),
                splitDirs,
                packageInfo.getOverlayDirs(),
                packageInfo.getApplicationInfo().sharedLibraryFiles,
                displayId,
                overrideConfiguration,
                compatInfo,
                classLoader));
        // ...
        return context;
 }
```

直接new出一个ContextImpl对象并且getInstance拿到ResourcesManager对象，然后创建Resources的工作会委托给单例对象ResourceManager，然后ComtextImpl会把ResourceManager创建的Resources保存在成员变量mResources中

## # ResourcesManager-> createBaseActivityResources

```java
public @Nullable Resources createBaseActivityResources(@NonNull IBinder activityToken,
            @Nullable String resDir,
            @Nullable String[] splitResDirs,
            @Nullable String[] overlayDirs,
            @Nullable String[] libDirs,
            int displayId,
            @Nullable Configuration overrideConfig,
            @NonNull CompatibilityInfo compatInfo,
            @Nullable ClassLoader classLoader) {
    
            final ResourcesKey key = new ResourcesKey(
                    resDir,
                    splitResDirs,
                    overlayDirs,
                    libDirs,
                    displayId,
                    overrideConfig != null ? new Configuration(overrideConfig) : null, // Copy
                    compatInfo);
            classLoader = classLoader != null ? classLoader : ClassLoader.getSystemClassLoader();
            synchronized (this) {
                // Force the creation of an ActivityResourcesStruct.
                getOrCreateActivityResourcesStructLocked(activityToken);
            }
    
            updateResourcesForActivity(activityToken, overrideConfig, displayId,
                    false /* movedToDifferentDisplay */);
            return getOrCreateResources(activityToken, key, classLoader);
        } 
    }
```

基于所有的资源目录、显示屏id等信息配置生成一个**ResourcesKey**

调用**getOrCreateActivityResourcesStructLocked**方法创建一个**ActivityResourcesStruct**添加到缓存中

调用**updateResourceForActivity**方法用**overideConfig**更新缓存中的**ActivityResourceStruct**

### ## ActivityResources结构

```java
private static class ActivityResources {
        @UnsupportedAppUsage
        private ActivityResources() {
        }
        public final Configuration overrideConfig = new Configuration();
        public final ArrayList<WeakReference<Resources>> activityResources = new ArrayList<>();
}
```

可见，其实**ActivityResources**就是与某个Activity相关的**Resources**集合以及配置信息的封装

### ##  ResourcesManager-> getOrCreateActivityResourcesStructLocked 

```java
private ActivityResources getOrCreateActivityResourcesStructLocked(
            @NonNull IBinder activityToken) {
        ActivityResources activityResources = mActivityResourceReferences.get(activityToken);
        if (activityResources == null) {
            activityResources = new ActivityResources();
            mActivityResourceReferences.put(activityToken, activityResources);
        }
        return activityResources;
    }
```

判断成员变量**mActivityResourceReferences**中是否有与token匹配的**ActivityResources**

如果存在直接返回

如果不存在创建一个并添加到缓存中

### ## ResourcesManager-> updateResourcesForActivity

```java
public void updateResourcesForActivity(@NonNull IBinder activityToken,
            @Nullable Configuration overrideConfig, int displayId,
            boolean movedToDifferentDisplay /* false */) {
       
            synchronized (this) {
                final ActivityResources activityResources =
                        getOrCreateActivityResourcesStructLocked(activityToken);

                if (Objects.equals(activityResources.overrideConfig, overrideConfig)
                        && !movedToDifferentDisplay/*false*/) {
                    // 如果activityResources中存的配置信息和传入的一致, 那就直接退出
                    return;
                }

                // 创建一个activityResources的旧的配置对象
                final Configuration oldConfig = new Configuration(activityResources.overrideConfig);

                // 更新activityResources中的配置对象
                if (overrideConfig != null) {
                    activityResources.overrideConfig.setTo(overrideConfig);
                } else {
                    activityResources.overrideConfig.unset();
                }

                final boolean activityHasOverrideConfig =
                        !activityResources.overrideConfig.equals(Configuration.EMPTY);

                // 重新设定与此Activity关联的每个资源的
                final int refCount = activityResources.activityResources.size();
                for (int i = 0; i < refCount; i++) {
                    WeakReference<Resources> weakResRef = activityResources.activityResources.get(
                            i);
                    // 获取Resources
                    Resources resources = weakResRef.get();
                    if (resources == null) {
                        continue;
                    }

                    // 根据Resource中ResourceImpl获取ResourceKey
                    final ResourcesKey oldKey = findKeyForResourceImplLocked(resources.getImpl());

                    // 构建一个内容与overrideConfig相同的配置对象
                    final Configuration rebasedOverrideConfig = new Configuration();
                    if (overrideConfig != null) {
                        rebasedOverrideConfig.setTo(overrideConfig);
                    }
                     
                    // 先用旧的配置信息和新的配置信息生成差量信息
                    // 把差量信息设置到rebasedOverrideConfig中
                    if (activityHasOverrideConfig && oldKey.hasOverrideConfiguration()) {
                        Configuration overrideOverrideConfig = Configuration.generateDelta(
                                oldConfig, oldKey.mOverrideConfiguration);
                        rebasedOverrideConfig.updateFrom(overrideOverrideConfig);
                    }

                    // 基于新的配置信息生成新的ResourcesKey
                    final ResourcesKey newKey = new ResourcesKey(oldKey.mResDir,
                            oldKey.mSplitResDirs,
                            oldKey.mOverlayDirs, oldKey.mLibDirs, displayId,
                            rebasedOverrideConfig, oldKey.mCompatInfo);

                    ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(newKey);
                    if (resourcesImpl == null) {
                        resourcesImpl = createResourcesImpl(newKey);
                        if (resourcesImpl != null) {
                            mResourceImpls.put(newKey, new WeakReference<>(resourcesImpl));
                        }
                    }

                    if (resourcesImpl != null && resourcesImpl != resources.getImpl()) {
                        resources.setImpl(resourcesImpl);
                    }
                }
    }
```

这里的Configuration配置信息其实就是和resource.arsc中的Res_Config类似

而与ResourceImpl相关的函数是：

* **findResourcesImplForKeyLocked**
* **createResourcesImpl**

总结起来这个方法的主要作用是：

* 更新**activityTokn**对应的**ActivityResources**中的配置信息，然后由于**ResourcesKey**会由配置信息组成，因此需要更新所有**ActivityResources**中的**ResourcesImpl**对应的**ResourcesKey**
* 创建**ResourceImpl**并把它添加到**ResourceKey**和**ResourceImpl**的映射缓存中

### ## ResourcesManager-> getOrCreateResources

```java
private @Nullable Resources getOrCreateResources(@Nullable IBinder activityToken,
            @NonNull ResourcesKey key, @NonNull ClassLoader classLoader) {
        synchronized (this) {
            if (activityToken != null) {
                // 应用层应用
                final ActivityResources activityResources =
                        getOrCreateActivityResourcesStructLocked(activityToken);
                ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                if (resourcesImpl != null) {
                    return getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                            resourcesImpl, key.mCompatInfo);
                }
            } else {
                // 系统应用
                ResourcesImpl resourcesImpl = findResourcesImplForKeyLocked(key);
                if (resourcesImpl != null) {
                    return getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
                }
            }
            
            ResourcesImpl resourcesImpl = createResourcesImpl(key);
            if (resourcesImpl == null) {
                return null;
            }

            mResourceImpls.put(key, new WeakReference<>(resourcesImpl));

            final Resources resources;
            if (activityToken != null) {
                resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader,
                        resourcesImpl, key.mCompatInfo);
            } else {
                resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
            }
            return resources;
        }
    }
```

 这里主要区分两种情况：

**情况1：**如果**activityToken**不为空，则说明是应用层的应用，执行步骤为

* 根据**activityToken**在缓存中获取**ActivityResources**
* 根据**ResourcesKey**在缓存中获取**ResourceImp**
* 调用**getOrCreateResourcesForActivityLocked**方法返回一个**Resource**对象

**情况2：**如果**activityToken**为空，则说明是系统应用，执行步骤为

* 少去**情况1**的第一不，直接执行步骤2、3
* 而**ActivityResources**的获取之后主要用于配置信息的处理

### ### ResourcesManager-> getOrCreateResourcesForActivityLocked

```java
private @NonNull Resources getOrCreateResourcesForActivityLocked(@NonNull IBinder activityToken,
            @NonNull ClassLoader classLoader, @NonNull ResourcesImpl impl,
            @NonNull CompatibilityInfo compatInfo) {
        final ActivityResources activityResources = getOrCreateActivityResourcesStructLocked(
                activityToken);

        final int refCount = activityResources.activityResources.size();
        for (int i = 0; i < refCount; i++) {
            WeakReference<Resources> weakResourceRef = activityResources.activityResources.get(i);
            Resources resources = weakResourceRef.get();

            if (resources != null
                    && Objects.equals(resources.getClassLoader(), classLoader)
                    && resources.getImpl() == impl) {
                return resources;
            }
        }

        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
                : new Resources(classLoader);
        resources.setImpl(impl);
        activityResources.activityResources.add(new WeakReference<>(resources));
        return resources;
    }
```

遍历**ActivityResources**中的**Resources**，找出**ResourcesImpl**相同的一项，找到就返回，假如找不到，就调用创建一个

## 小结

源码有点多有点乱，往下继续深挖之前，先组织和小结一下**ContextImpl**、**Resources**、**ResourcesImpl**、**ResourcesManager**的作用及关系

* 在**ContextImpl**初始化的时候会创建一个**Resources**对象并用成员变量**mResources**保存起来
* **ContextImpl**中的**Resources**的创建工作是委托给**ResourceManager**实现的
* 而**ResourceManager**则会创建根据**ContextImpl**传过来的信息(主要是资源目录, 显示id)生成**ResourcesKey**，这个**ResourcesKey**是用于作为**ResourcesImpl**缓存的索引
* 而**ResourceManager**还会根据**ContextImpl**传来的的**activityToken**创建一个**ActivityResources**，主要作用是保存与这个**Activity**相关的所有**Resource**以及配置信息
* 而从**ResourcesManager**向**ContextImpl**返回的**Resource**对象规则为：
  * 通过**activityToken**获取**Resource**列表
  * 遍历**Resources**列表拿到和**ResourcesKey**对应缓存中**ResourcesImpl**相同的一个
  * 如果找不到就创建一个**Resources**绑定**ResourcesImpl**，并返回

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/Resources.png)

而**Resources**是**ResoucesImpl**的代理，**Resources**所执行的操作都是交由**ResourcesImpl**来完成，除此之外**Resources**还管理着系统资源的接口，用于对外提供访问系统资源的方式

而在代码设计上这个部分有一个特点，那就是创建之后的对象都会放在缓存中，然后其他方法需要用到这些对象的时候只需要从缓存中获取就可以了

因此下面继续跟踪资源管理系统初始化就可以从**ResourcesImpl**的开始

## # ResourcesManager-> createResourcesImpl

```java
 private @NonNull ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key) {
        final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
        daj.setCompatibilityInfo(key.mCompatInfo);
        final AssetManager assets = createAssetManager(key);
        final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
        final Configuration config = generateConfig(key, dm);
        final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);
        return impl;
    }
```

## # ResourcesManager-> createAssetManager

```java
 protected @Nullable AssetManager createAssetManager(@NonNull final ResourcesKey key) {
        final AssetManager.Builder builder = new AssetManager.Builder();

        // resDir can be null if the 'android' package is creating a new Resources object.
        // This is fine, since each AssetManager automatically loads the 'android' package
        // already.
        if (key.mResDir != null) {
            try {
                builder.addApkAssets(loadApkAssets(key.mResDir, false /*sharedLib*/,
                        false /*overlay*/));
            } catch (IOException e) {
                Log.e(TAG, "failed to add asset path " + key.mResDir);
                return null;
            }
        }

        if (key.mSplitResDirs != null) {
            for (final String splitResDir : key.mSplitResDirs) {
                try {
                    builder.addApkAssets(loadApkAssets(splitResDir, false /*sharedLib*/,
                            false /*overlay*/));
                } catch (IOException e) {
                    Log.e(TAG, "failed to add split asset path " + splitResDir);
                    return null;
                }
            }
        }

        if (key.mOverlayDirs != null) {
            for (final String idmapPath : key.mOverlayDirs) {
                try {
                    builder.addApkAssets(loadApkAssets(idmapPath, false /*sharedLib*/,
                            true /*overlay*/));
                } catch (IOException e) {
                    Log.w(TAG, "failed to add overlay path " + idmapPath);

                    // continue.
                }
            }
        }

        if (key.mLibDirs != null) {
            for (final String libDir : key.mLibDirs) {
                if (libDir.endsWith(".apk")) {
                    // Avoid opening files we know do not have resources,
                    // like code-only .jar files.
                    try {
                        builder.addApkAssets(loadApkAssets(libDir, true /*sharedLib*/,
                                false /*overlay*/));
                    } catch (IOException e) {
                        Log.w(TAG, "Asset path '" + libDir +
                                "' does not exist or contains no resources.");
                        // continue.
                    }
                }
            }
        }

        return builder.build();
    }
```

**AssetManager**会把资源都加载进来，而从代码逻辑能看出来，主要加载的逻辑为：

* 调用**loadApkAssets**创建**ApkAssets**
* 把**ApkAssets**添加到**AssetManager.Builder**中，最后生成**AssetManager**

### ## ResourcesManager-> loadApkAssets

```java
private @NonNull ApkAssets loadApkAssets(String path, boolean sharedLib, boolean overlay)
            throws IOException {
        final ApkKey newKey = new ApkKey(path, sharedLib, overlay);
        ApkAssets apkAssets = mLoadedApkAssets.get(newKey);
        if (apkAssets != null) {
            return apkAssets;
        }

        // Optimistically check if this ApkAssets exists somewhere else.
        final WeakReference<ApkAssets> apkAssetsRef = mCachedApkAssets.get(newKey);
        if (apkAssetsRef != null) {
            apkAssets = apkAssetsRef.get();
            if (apkAssets != null) {
                mLoadedApkAssets.put(newKey, apkAssets);
                return apkAssets;
            } else {
                // Clean up the reference.
                mCachedApkAssets.remove(newKey);
            }
        }

        if (overlay) {
            apkAssets = ApkAssets.loadOverlayFromPath(overlayPathToIdmapPath(path),
                    false /*system*/);
        } else {
            apkAssets = ApkAssets.loadFromPath(path, false /*system*/, sharedLib);
        }
        mLoadedApkAssets.put(newKey, apkAssets);
        mCachedApkAssets.put(newKey, new WeakReference<>(apkAssets));
        return apkAssets;
    }
```

根据资源的路径，shareLib，Overlay生成一个**ApkKey**

从**mLoadedApkAssets**中根据**ApkKey**获取一个**ApkAssets**，如果存在直接返回

如果上一步获取不成功，就会从**mCachedApkAssets**中根据**ApkKey**获取，如果存在，就添加到**mLoadedApkAssets**中并从**mCachedApkAssets**中移除，然后返回

如果缓存中没有就直接创建一个，然后添加到**mLoadedApkAssets**和**mCachedApkAssets**中

**设计思想：**

所有资源目录都会生成一个**ApkAssets**，然后做一个二级缓存

* **第一级缓存：** **mLoadedApkAssets**保存这所有已经加载的了**ApkAssets**的强引用 

* **第二级缓存：** **mCachedApkAssets**保存这所有加载过的**ApkAssets**的弱引用

这种设计的目的就是把缓存拆分两部分：

* **活跃缓存：** 活跃缓存持有强引用避免GC销毁
* **非活跃缓存：**非活跃活跃缓存则持有弱引用，就算GC销毁了也不会有什么问题 

### ### ApkAssets的加载 

```java
  @GuardedBy("this") private final long mNativePtr;
  @GuardedBy("this") private final StringBlock mStringBlock;

   public static @NonNull ApkAssets loadOverlayFromPath(@NonNull String idmapPath, boolean system)
            throws IOException {
        return new ApkAssets(idmapPath, system, false /*forceSharedLibrary*/, true /*overlay*/);
    }

    public static @NonNull ApkAssets loadFromPath(@NonNull String path, boolean system,
            boolean forceSharedLibrary) throws IOException {
        return new ApkAssets(path, system, forceSharedLibrary, false /*overlay*/);
    }

    public static @NonNull ApkAssets loadFromPath(@NonNull String path, boolean system)
            throws IOException {
        return new ApkAssets(path, system, false /*forceSharedLib*/, false /*overlay*/);
    }

    private ApkAssets(@NonNull String path, boolean system, boolean forceSharedLib, boolean overlay)
            throws IOException {
        mNativePtr = nativeLoad(path, system, forceSharedLib, overlay);
        mStringBlock = new StringBlock(nativeGetStringBlock(mNativePtr), true /*useSparse*/);
    }
```

ApkAssets可以通过两种方式创建：

- **ApkAssets.loadOverlayFromPath** 当apk使用到了额外重叠的资源目录对应的**ApkAsset**
- **ApkAssets.loadFromPath** 当apk使用一般的资源，比如的value资源，第三方资源库等创建对应的**ApkAsset**

 **ApkAssets**的实际创建工作就委托给了**Native**层：

* 调用**nativeLoad**方法创建一个C++层的**ApkAssets**，然后只要用long把地址保存起来就可以了
* 调用**nativeGetStringBlock**初始化Java层的**StringBlock**，逻辑也类似

### ### android_content_res_ApkAssets.cpp-> NativeLoad

```c++
static jlong NativeLoad(JNIEnv* env, jclass /*clazz*/, jstring java_path, jboolean system,
                        jboolean force_shared_lib, jboolean overlay) {
  ScopedUtfChars path(env, java_path);
  // ......
  std::unique_ptr<const ApkAssets> apk_assets;
  if (overlay) {
    apk_assets = ApkAssets::LoadOverlay(path.c_str(), system);
  } else if (force_shared_lib) {
    apk_assets = ApkAssets::LoadAsSharedLibrary(path.c_str(), system);
  } else {
    apk_assets = ApkAssets::Load(path.c_str(), system);
  }

  if (apk_assets == nullptr) {
    // ......
    return 0;
  }
  return reinterpret_cast<jlong>(apk_assets.release());
}
```

这个native方法根据3种情况创建**apk_assets**：

* **LoadOverlay** 加载重叠资源
* **LoadAsSharedLibrary** 加载第三方库资源
* **Load** 加载一般的资源

重叠资源与我们平常的换肤不一样，输入framework层的资源包替换，如果想把Android相关资源替换掉，此时在overlay的文件夹中会包含这个apk，这个apk中只有资源没有dex，并且把相关能替换的id写在某个问价上，而在**AssetManager**初始化的时候就会根据这个id替换所有资源

 ### ### ApkAssets-> Load

```java
static const std::string kResourcesArsc("resources.arsc");

std::unique_ptr<const ApkAssets> ApkAssets::Load(const std::string& path, bool system) {
  return LoadImpl({} /*fd*/, path, nullptr, nullptr, system, false /*load_as_shared_library*/);
}

std::unique_ptr<const ApkAssets> ApkAssets::LoadImpl(
    unique_fd fd, const std::string& path, std::unique_ptr<Asset> idmap_asset,
    std::unique_ptr<const LoadedIdmap> loaded_idmap, bool system, bool load_as_shared_library) {
  ::ZipArchiveHandle unmanaged_handle;
  int32_t result;
  // 1  
  if (fd >= 0) {
    result =
        ::OpenArchiveFd(fd.release(), path.c_str(), &unmanaged_handle, true /*assume_ownership*/);
  } else {
    result = ::OpenArchive(path.c_str(), &unmanaged_handle);
  }
  // 用unique_ptr包裹handle，实现自动关闭
  std::unique_ptr<ApkAssets> loaded_apk(new ApkAssets(unmanaged_handle, path));

  // 2
  ::ZipString entry_name(kResourcesArsc.c_str());
  ::ZipEntry entry;
  result = ::FindEntry(loaded_apk->zip_handle_.get(), entry_name, &entry);
  if (result != 0) {
    // ......
    loaded_apk->loaded_arsc_ = LoadedArsc::CreateEmpty();
    return std::move(loaded_apk);
  }
    
  // 3  
  loaded_apk->resources_asset_ = loaded_apk->Open(kResourcesArsc, Asset::AccessMode::ACCESS_BUFFER);
  loaded_apk->idmap_asset_ = std::move(idmap_asset);
    
  // 4  
  const StringPiece data(
      reinterpret_cast<const char*>(loaded_apk->resources_asset_->getBuffer(true /*wordAligned*/)),
      loaded_apk->resources_asset_->getLength());
  loaded_apk->loaded_arsc_ =
      LoadedArsc::Load(data, loaded_idmap.get(), system, load_as_shared_library);

  return std::move(loaded_apk);
}
```

**步骤1：**如果有fd文件描述符则用调用**OpenArchiveFd**打开Zip文件，如果没有fd则调用**OpenArchive**打开Zip文件，总的来说说就是打开Zip文件

**步骤2：**通过**FindEntry**函数，寻找apk包中的**resource.arsc**文件 

**步骤3：**读取apk包中的**resource.arsc**文件，读取里面包含的id相关的map，以及资源asset文件夹中 

**步骤4：**生成**StringPiece**对象，通过**LoadedArsc::Load**读取其中的数据 

 ### #### ApkAssets-> Open

```c++
std::unique_ptr<Asset> ApkAssets::Open(const std::string& path, Asset::AccessMode mode) const {
  CHECK(zip_handle_ != nullptr);

  ::ZipString name(path.c_str());
  ::ZipEntry entry;
  int32_t result = ::FindEntry(zip_handle_.get(), name, &entry);
  if (result != 0) {
    return {};
  }

  if (entry.method == kCompressDeflated) {
    std::unique_ptr<FileMap> map = util::make_unique<FileMap>();
    if (!map->create(path_.c_str(), ::GetFileDescriptor(zip_handle_.get()), entry.offset,
                     entry.compressed_length, true /*readOnly*/)) {
      // ......
      return {};
    }

    std::unique_ptr<Asset> asset =
        Asset::createFromCompressedMap(std::move(map), entry.uncompressed_length, mode);
    if (asset == nullptr) {
      // ......
      return {};
    }
    return asset;
  } else {
    std::unique_ptr<FileMap> map = util::make_unique<FileMap>();
    if (!map->create(path_.c_str(), ::GetFileDescriptor(zip_handle_.get()), entry.offset,
                     entry.uncompressed_length, true /*readOnly*/)) {
      // ......
      return {};
    }

    std::unique_ptr<Asset> asset = Asset::createFromUncompressedMap(std::move(map), mode);
    if (asset == nullptr) {
      // ......
      return {};
    }
    return asset;
  }
}
```

这个函数会把ZipEntry传进来，然后判断这个ZipEntry是否经过压缩：

* **如果经过压缩：**
  * 先通过FileMap把ZipEntry通过mmap映射到虚拟内存中
  * 再通过Asset::createFromCompressedMap通过_CompressedAsset::openChunk拿到StreamingZipInflater，返回_CompressedAsset对象
  * CompressedAsset是Asset的子类，在openChunk创建了StreamingZipInflater之后用成员变量保存起来，但是不会马上进行解压操作，而是等到第一次读操作执行
* **没有经过压缩：**
  * 通过FileMap把ZipEntry通过mmap映射到虚拟内存中
  * 最后Asset::createFromUncompressedMap，返回FileAsset对象 

resource.arsc并没有在apk中没有压缩，因此走的下面，直接返回对应的FileAsset

由这个逻辑可见，实际上对于压缩文件和非压缩文件，加载的流程类似，只不过压缩文件的加载，还需要记录一些和压缩相关的信息如：压缩文件的偏移量,长度,压缩后大小以及压 缩前大小等信息，然后里有加载过程中创建好的StreamingZipInflater在第一次读的时候进行解压操作

Android为了加速资源的加载速度，并不是直接通过File读写操作读取资源信息。而是通过FileMap的方式，也就是mmap把文件地址映射到虚拟内存中，时刻准备读写，这么做的好处，就是mmap回返回文件的地址，可以对文件进行操作，节省系统调用的开销，坏处就是mmap会映射到虚拟内存中，是的虚拟内存增大 

### ### LoadedArsc-> Load

```c++
std::unique_ptr<const LoadedArsc> LoadedArsc::Load(const StringPiece& data,
                                                   const LoadedIdmap* loaded_idmap, bool system,
                                                   bool load_as_shared_library) {

  std::unique_ptr<LoadedArsc> loaded_arsc(new LoadedArsc());
  loaded_arsc->system_ = system;

  ChunkIterator iter(data.data(), data.size());
  while (iter.HasNext()) {
    const Chunk chunk = iter.Next();
    switch (chunk.type()) {
      case RES_TABLE_TYPE:
        if (!loaded_arsc->LoadTable(chunk, loaded_idmap, load_as_shared_library)) {
          return {};
        }
        break;
        // ......
    }
  }
  // ......
}
```

所有zip的chunk解析出来后，迭代寻找resource.arsc文件的标志头RES_TABLE_TYPE，也就arsc文件的头两个字节，然后执行的工作就是根据resources.arsc的文件结构把它读取出来，做成一个内存中的数据结构供后面使用

主要为LoadedArsc：

​          -  gobal_string_pool_ 全局字符串资源池

​          -  LoadedPackage package 数据对象

当解析完resources.arsc文件之后就会把地址返回给java层

### ### ApkAssers的加载

```java
    private ApkAssets(@NonNull String path, boolean system, boolean forceSharedLib, boolean overlay)
            throws IOException {
        Preconditions.checkNotNull(path, "path");
        mNativePtr = nativeLoad(path, system, forceSharedLib, overlay);
        mStringBlock = new StringBlock(nativeGetStringBlock(mNativePtr), true /*useSparse*/);
    }
```

执行完了native层ApkAssets的创建，会再走一次native层，目标是拿到StringBlock

### ### android_content_res_ApkAssets.cpp-> NativeGetStringBlock

```c++
static jlong NativeGetStringBlock(JNIEnv* /*env*/, jclass /*clazz*/, jlong ptr) {
  const ApkAssets* apk_assets = reinterpret_cast<const ApkAssets*>(ptr);
  return reinterpret_cast<jlong>(apk_assets->GetLoadedArsc()->GetStringPool());
}
```

### ### LoadedArsc-> GetStringPool

```c++
inline const ResStringPool* GetStringPool() const {
    return &global_string_pool_;
 }
```

stringBlock的初始化其实就再走一次native层，把存在LoadedArsc中的global_string_pool_拿出来

此时ApkAssets就持有了两个native对象，一个是native层的ApkAssets，一个是native层的保存的arsc文件解析出来全局常量池

## # AssetManager.Builder 

```java
public static class Builder {
   
   private ArrayList<ApkAssets> mUserApkAssets = new ArrayList<>();

   public AssetManager.Builder addApkAssets() {
       mUserApkAssets.add(apkAssets);
       return this;
   }
    
   public AssetManager build() {
       // Retrieving the system ApkAssets forces their creation as well.
       final ApkAssets[] systemApkAssets = getSystem().getApkAssets();

       final int totalApkAssetCount = systemApkAssets.length + mUserApkAssets.size();
       final ApkAssets[] apkAssets = new ApkAssets[totalApkAssetCount];

       System.arraycopy(systemApkAssets, 0, apkAssets, 0, systemApkAssets.length);

       final int userApkAssetCount = mUserApkAssets.size();
       for (int i = 0; i < userApkAssetCount; i++) {
            apkAssets[i + systemApkAssets.length] = mUserApkAssets.get(i);
       }

       // Calling this constructor prevents creation of system ApkAssets, which we took care
       // of in this Builder.
       final AssetManager assetManager = new AssetManager(false /*sentinel*/);
       assetManager.mApkAssets = apkAssets;
       AssetManager.nativeSetApkAssets(assetManager.mObject, apkAssets, false /*invalidateCaches*/);
       return assetManager;      
    }
}   
```

**AssetManager**的创建使用的是builder模式，流程为：

* 拿到管理系统资源的**AssetManager**，然后拿到其中ApkAsset数组
* 然后把系统资源的ApkAsset和应用自身的ApkAsset合并成一个新数组
* 然后就是直接new出一个**AssetManager**，把ApkAsset赋值给这个AssetManager
* 最后调用nativeSetApkAssets给native层的**AssetManager2**赋值

### ## AssetManager系统资源初始化

```java
public static AssetManager getSystem() {
        synchronized (sSync) {
            createSystemAssetsInZygoteLocked();
            return sSystem;
        }
 }

private static void createSystemAssetsInZygoteLocked() {
        if (sSystem != null) {
            return;
        }
    
         final ArrayList<ApkAssets> apkAssets = new ArrayList<>();
         apkAssets.add(ApkAssets.loadFromPath("/system/framework/framework-res.apk", true /*system*/));
         loadStaticRuntimeOverlays(apkAssets);

         sSystemApkAssetsSet = new ArraySet<>(apkAssets);
         sSystemApkAssets = apkAssets.toArray(new ApkAssets[apkAssets.size()]);
         sSystem = new AssetManager(true /*sentinel*/);
         sSystem.setApkAssets(sSystemApkAssets, false /*invalidateCaches*/);
      }
  }
```

先构建一个静态的AssetManager，这个AssetManager只管理一个资源包**/system/framework/framework-res.apk**

然后调用**loadStaticRuntimeOverlays**方法根据 **/data/resource-cache/overlays.list**的复写资源文件，把需要重叠的资源覆盖在系统apk上

### ## AssetManager构造函数 

```java
private AssetManager(boolean sentinel) {
        mObject = nativeCreate();
 }
```

AssetManager的构造函数就是调用nativeCreate函数，而到了native层也只是直接new出对应的AssetManager2对象

### ## AssetManager2.cpp-> NativeSetApkAssets 

把java层的数据转变成C++层的Vector，然后设置到**AssetManager2**中，当然Vector中的**ApkAssets**是指向C++层的实际对象

然后转调AssetManager2的**SetApkAssets**

###  ##  AssetManager2.cpp-> SetApkAssets

```java
bool AssetManager2::SetApkAssets(const std::vector<const ApkAssets*>& apk_assets,
                                 bool invalidate_caches) {
  apk_assets_ = apk_assets;
  BuildDynamicRefTable();
  // ......  
  return true;
}
```

**BuildDynamicRefTable**构建动态的资源引用表

**RebuildFilterList**构建过滤后的配置列表

**InvalidateCaches**刷新缓存

 ### ### AssetManager2-> BuildDynamicRefTable

```c++
std::vector<PackageGroup> package_groups_;

void AssetManager2::BuildDynamicRefTable() {
  package_groups_.clear();
  package_ids_.fill(0xff);

  // 0x01 is reserved for the android package.
  int next_package_id = 0x02;
  const size_t apk_assets_count = apk_assets_.size();
  for (size_t i = 0; i < apk_assets_count; i++) {
    const LoadedArsc* loaded_arsc = apk_assets_[i]->GetLoadedArsc();
  
    for (const std::unique_ptr<const LoadedPackage>& package : loaded_arsc->GetPackages()) {
      // Get the package ID or assign one if a shared library.
      int package_id;
      if (package->IsDynamic()) {
        package_id = next_package_id++;
      } else {
        package_id = package->GetPackageId();
      }

      // Add the mapping for package ID to index if not present.
      uint8_t idx = package_ids_[package_id];
      if (idx == 0xff) {
        package_ids_[package_id] = idx = static_cast<uint8_t>(package_groups_.size());
        package_groups_.push_back({});
        DynamicRefTable& ref_table = package_groups_.back().dynamic_ref_table;
        ref_table.mAssignedPackageId = package_id;
        ref_table.mAppAsLib = package->IsDynamic() && package->GetPackageId() == 0x7f;
      }
      PackageGroup* package_group = &package_groups_[idx];

      // Add the package and to the set of packages with the same ID.
      package_group->packages_.push_back(ConfiguredPackage{package.get(), {}});
      package_group->cookies_.push_back(static_cast<ApkAssetsCookie>(i));
  
      // Add the package name -> build time ID mappings.
      for (const DynamicPackageEntry& entry : package->GetDynamicPackageMap()) {
        String16 package_name(entry.package_name.c_str(), entry.package_name.size());
        package_group->dynamic_ref_table.mEntries.replaceValueFor(
            package_name, static_cast<uint8_t>(entry.package_id));
      }
    }
  }

  // Now assign the runtime IDs so that we have a build-time to runtime ID map.
  const auto package_groups_end = package_groups_.end();
  for (auto iter = package_groups_.begin(); iter != package_groups_end; ++iter) {
    const std::string& package_name = iter->packages_[0].loaded_package_->GetPackageName();
    for (auto iter2 = package_groups_.begin(); iter2 != package_groups_end; ++iter2) {
      iter2->dynamic_ref_table.addMapping(String16(package_name.c_str(), package_name.size()),
                                          iter->dynamic_ref_table.mAssignedPackageId);
    }
  }
}
```

这里有点小坑，这里处于AssetManager2的所有apk_assers遍历之中，会从apk_assers中拿到之前构建时候解析好的LoadedArsc，然后从中拿出package，最后划分两种情况进行赋值给局部变量packageId：

* share library：这种情况下会在最开始创建ApkAssets的时候给标记为forceSharedLibrary为true，这种情况下，packageId会为0，所以需要从0x02开始给这个packageId赋值
* 一般的应用package，在解析的时候指定了packageId为0x7f，所以这个时候，packageId就是原来的load的时候指定的packageId，也就是0x7f，由于给package_ids fill了初值0xff，所以代表了未初始化的情况，这个时候会把package_group的大小赋值到package_ids中保存起来

把一个LoadedArsc中的所有package添加到package_group中，并且为每一个index设置一个cookie，这个cookie本质上是一个int类型，随着package的增大而增加 

 ### ### DynamicRefTable.cpp-> addMapping

```c++
status_t DynamicRefTable::addMapping(const String16& packageName, uint8_t packageId)
{
    ssize_t index = mEntries.indexOfKey(packageName);
    if (index < 0) {
        return UNKNOWN_ERROR;
    }
    mLookupTable[mEntries.valueAt(index)] = packageId;
    return NO_ERROR;
}
```

### ### AssetManager2-> RebuildFilterList

```c++
void AssetManager2::RebuildFilterList() {
  for (PackageGroup& group : package_groups_) {
    for (ConfiguredPackage& impl : group.packages_) {
      // Destroy it.
      impl.filtered_configs_.~ByteBucketArray();

      // Re-create it.
      new (&impl.filtered_configs_) ByteBucketArray<FilteredConfigGroup>();

      // Create the filters here.
      impl.loaded_package_->ForEachTypeSpec([&](const TypeSpec* spec, uint8_t type_index) {
        FilteredConfigGroup& group = impl.filtered_configs_.editItemAt(type_index);
        const auto iter_end = spec->types + spec->type_count;
        for (auto iter = spec->types; iter != iter_end; ++iter) {
          ResTable_config this_config;
          this_config.copyFromDtoH((*iter)->config);
          if (this_config.match(configuration_)) {
            group.configurations.push_back(this_config);
            group.types.push_back(*iter);
          }
        }
      });
    }
  }
}
```

cached_bags_实际上缓存着过去生成过资源id，如果需要则会清除，一般这种情况如AssetManager配置发生变化都会清除一下避免干扰cached_bags

 而这几个步骤的目的主要是为了让native层的AssetManager变相通过PackageGroup持有apk中的资源

## 总结

所有类的关系结构：

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/relation%20of%20resources.png)

**Java层**

* **ResourcesImpl**是对外提供资源操作的Resources的真正实现，**Resources**只是一个代理
* **ResourcesImpl**中会有一个**AssetManager**，**AssetManager**用于维护一个列表存储并管理**ApkAssets**
* **ApkAssets**表示一个资源文件夹单位，数据及实现主要在native层，Java层维护两个指向native层的指针

**C++层**

* **ApkAssets**表示一个资源文件单位，主要存储了resources.arsc资源文件映射对象及解析后数据结构，它包含两个重要的数据结构：

  * **resources_asset_** ：象征着一个本质上是resources.arsc zip资源FileMap的Asset 
  * **loaded_arsc_：** ，一个**LoadedArsc**，是resource.arsc解析资源后生成的对象

* **LoadedArsc**表示resources.arsc的解析数据结构，它包含两个重要的数据结构：

  * **global_string_pool_** 全局字符串资源池
  * **LoadedPackage package** 数据对象

* **LoadedPackage**里面有着大量的资源对象相关信息，以及真实数据，其中也包含几个很重要的数据结构：

  * **type_string_pool_** 资源类型字符串
  * **key_string_pool_** 资源项字符串
  * **type_specs_** 保存着所有资源类型和资源项的映射关系
  * **dynamic_package_map_** 保存第三方资源库的包名和packageId的映射关系

* **AssetManager2** Java层的AssetManager中存有AssetManager2的地址值，AssetManager2中也保存了**ApkAssets**的列表并对其维护，而**AssetManager2**还进行了优化：

  * 保存着多个**PackageGroup**对象(内含ConfiguredPackage)，里面包含着所有package数据块

  * 构建动态资源表，放在**package_group**中，为了解决packageID运行时和编译时冲突问题

  * 提前筛选出符合当前环境的资源配置到**FilteredConfigGroup**，为了可以快速访问

  * 缓存已经访问过的BagID，也就是完整的资源ID

资源管理系统初始化中体现的Android读取优化措施：

* 在**ResourcesManager**中通过activityToken为Key缓存**ActivityResources**，缓存了该Activity所关联的所有的Resources对象的弱引用
* 在**ResourcesManager**中通过与资源相关信息封装成**ResourcesKey**作为Key，缓存**ResourcesImpl**的弱引用
* **ApkAssets**在内存中中的缓存，缓存拆成两部分，**mLoadedApkAssets**已经加载的活跃**ApkAssets**，**mCacheApkAssets**已经加载了但是不活跃的**ApkAssets** 
* 为了能够快速查找符合当前环境配置的资源(屏幕密度，语言环境等)，同样在过滤构建资源阶段，有一个**FilteredConfigGroup**对象，提供快速查找 
* 缓存已经访问过的**BagID**，也就是完整的资源ID 

 ![](https://upload-images.jianshu.io/upload_images/9880421-1c62a2722ec6562c.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

 

 

 

​     

​     

​     

​     





参考资料：

[Android应用程序资源管理器（Asset Manager）的创建过程分析](<https://blog.csdn.net/Luoshengyang/article/details/8791064>)

[Android 重学系列 资源管理系统 资源的初始化加载(上)](<https://www.jianshu.com/p/817a787910f2>)

[Android 重学系列 资源管理系统 资源的初始化加载(下)](<https://www.jianshu.com/p/02a2539890dc>)

[Android系统加载资源文件源码分析](<http://gttiankai.github.io/2019/06/02/android%E7%B3%BB%E7%BB%9F%E5%8A%A0%E8%BD%BD%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/> )