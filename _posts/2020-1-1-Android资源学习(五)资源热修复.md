---
layout:     post   				    
title:      Android资源学习(五)资源热修复
subtitle:   Android资源学习系列   #副标题
date:       2020-1-1		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-keybord.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android资源学习系列
---

# Android资源学习(五)资源热修复

## 前言

本文主要结合Android9.0资源编译，资源查找的流程，分析资源热修复的处理和实现，一开始找源码找得有点小累，因为在Android Code Search中怎么搜都搜不到，后来就只能去google git看了，源码的目录就在

[android](https://android.googlesource.com/?format=HTML) / [platform](https://android.googlesource.com/platform/) / [tools](https://android.googlesource.com/platform/tools/) / [base](https://android.googlesource.com/platform/tools/base/) / [refs/tags/gradle_3.4.0](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0) / [.](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/) / [instant-run](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run) / [instant-run-server](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server) / [src](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src) / [main](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main?autodive=0) / [java](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java?autodive=0) / [com](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java/com?autodive=0) / [android](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java/com/android?autodive=0) / [tools](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java/com/android/tools?autodive=0) / [ir](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java/com/android/tools/ir?autodive=0) / [server](https://android.googlesource.com/platform/tools/base/+/refs/tags/gradle_3.4.0/instant-run/instant-run-server/src/main/java/com/android/tools/ir/server) / 

## instant-run

> 源码基于gradle_3.4.0

### MonkeyPatcher-> monkeyPatchExistingResources

```java
public static void monkeyPatchExistingResources(@Nullable Context context,
                                                    @Nullable String externalResourceFile,
                                                    @Nullable Collection<Activity> activities) {
        if (externalResourceFile == null) {
            return;
        }
        try {
            // 步骤1：创建一个新的AssetManager，并通过反射调用addAssetPath添加sdcard上的新资源包
            AssetManager newAssetManager = AssetManager.class.getConstructor().newInstance();
            Method mAddAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
            mAddAssetPath.setAccessible(true);
            if (((Integer) mAddAssetPath.invoke(newAssetManager, externalResourceFile)) == 0) {
                throw new IllegalStateException("Could not create new AssetManager");
            }
            // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
            // in L, so we do it unconditionally.
            Method mEnsureStringBlocks = AssetManager.class.getDeclaredMethod("ensureStringBlocks");
            mEnsureStringBlocks.setAccessible(true);
            mEnsureStringBlocks.invoke(newAssetManager);
            if (activities != null) {
                // 遍历所有的activity，然后它们的Resources对象中mAssets替换掉
                for (Activity activity : activities) {
                    Resources resources = activity.getResources();
                    try {
                        Field mAssets = Resources.class.getDeclaredField("mAssets");
                        mAssets.setAccessible(true);
                        mAssets.set(resources, newAssetManager);
                    } catch (Throwable ignore) {
                        Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                        mResourcesImpl.setAccessible(true);
                        Object resourceImpl = mResourcesImpl.get(resources);
                        Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                        implAssets.setAccessible(true);
                        implAssets.set(resourceImpl, newAssetManager);
                    }
                }       
            // ......       
            // Iterate over all known Resources objects
            // 处理系统应用的Resources    
            Collection<WeakReference<Resources>> references;
            if (SDK_INT >= KITKAT) {
                // Find the singleton instance of ResourcesManager
                Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
                Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance");
                mGetInstance.setAccessible(true);
                Object resourcesManager = mGetInstance.invoke(null);
                try {
                    Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
                    fMActiveResources.setAccessible(true);
                    @SuppressWarnings("unchecked")
                    ArrayMap<?, WeakReference<Resources>> arrayMap =
                            (ArrayMap<?, WeakReference<Resources>>) fMActiveResources.get(resourcesManager);
                    references = arrayMap.values();
                } catch (NoSuchFieldException ignore) {
                    Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
                    mResourceReferences.setAccessible(true);
                    //noinspection unchecked
                    references = (Collection<WeakReference<Resources>>) mResourceReferences.get(resourcesManager);
                }
            } else {
                Class<?> activityThread = Class.forName("android.app.ActivityThread");
                Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
                fMActiveResources.setAccessible(true);
                Object thread = getActivityThread(context, activityThread);
                @SuppressWarnings("unchecked")
                HashMap<?, WeakReference<Resources>> map =
                        (HashMap<?, WeakReference<Resources>>) fMActiveResources.get(thread);
                references = map.values();
            }
            for (WeakReference<Resources> wr : references) {
                Resources resources = wr.get();
                if (resources != null) {
                    // Set the AssetManager of the Resources instance to our brand new one
                    try {
                        Field mAssets = Resources.class.getDeclaredField("mAssets");
                        mAssets.setAccessible(true);
                        mAssets.set(resources, newAssetManager);
                    } catch (Throwable ignore) {
                        Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                        mResourcesImpl.setAccessible(true);
                        Object resourceImpl = mResourcesImpl.get(resources);
                        Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                        implAssets.setAccessible(true);
                        implAssets.set(resourceImpl, newAssetManager);
                    }
                    resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
                }
            }
        } catch (Throwable e) {
            throw new IllegalStateException(e);
        }
    }
```

**主要流程：**

* 构建一个新的AssetManager，并通过反射调用**addAssetPath**，把这个完整的新资源包加到AssetManager中，得到一个含有所有新资源的AssetManager
* 找到所有之前引用到原有AssetManager的地方，通过反射，把引用处替换为AssetManager

**解读：** 大量的代码都是在处理兼容性问题和找到所有AssetManager的引用处，所有逻辑的关键就是**addAssetPath**，

在9.0的源码中这个方法已经标记为废弃方法

### AssetManager-> addAssetPath

```java
 public int addAssetPath(String path) {
        return addAssetPathInternal(path, false /*overlay*/, false /*appAsLib*/);
 }
```

### AssetManager-> addAssetPathInternal

```java
private int addAssetPathInternal(String path, boolean overlay, boolean appAsLib) {
        synchronized (this) {
            ensureOpenLocked();
            final int count = mApkAssets.length;

            // See if we already have it loaded.
            for (int i = 0; i < count; i++) {
                if (mApkAssets[i].getAssetPath().equals(path)) {
                    return i + 1;
                }
            }

            final ApkAssets assets;
            try {
                if (overlay) {
                    // TODO(b/70343104): This hardcoded path will be removed once
                    // addAssetPathInternal is deleted.
                    final String idmapPath = "/data/resource-cache/"
                            + path.substring(1).replace('/', '@')
                            + "@idmap";
                    assets = ApkAssets.loadOverlayFromPath(idmapPath, false /*system*/);
                } else {
                    assets = ApkAssets.loadFromPath(path, false /*system*/, appAsLib);
                }
            } catch (IOException e) {
                return 0;
            }

            mApkAssets = Arrays.copyOf(mApkAssets, count + 1);
            mApkAssets[count] = assets;
            nativeSetApkAssets(mObject, mApkAssets, true);
            invalidateCachesLocked(-1);
            return count + 1;
        }
    }
```

**方法流程**

* 把加载的工作委托给ApkAsset，然后把ApkAsset添加到数组中，然后同样设置到native层中

**解读**

* 对比Android6.0，可以看出来google在Android9.0把**addAssetPath**方法改了，让包裹在外面的**addAssetPath**标记为废弃，然后转调**addAssetPath**了不影响反射调用的地方，让**addAssetPath**转调**addAssetPathInternal**，而实际就是换汤不换药，实际处理和**AssetManager**初始化的工作类似

### 小结

instant-run的资源热修复方案相对比较简单，主要还是分为两个大步骤：

**补丁包的生成：**生成全量的不补丁

**补丁包下发成功的处理：**

* 反射创建一个新的**AssetManager**
* 反射调用**AssetManager**的**addAssetPath**方法把补丁包加载进去
* 通过反射替换所有Resources的**AssetManager**

## Tinker

### ResDiffDecoder-> onAllPatchesStart

```java
 @Override
    public void onAllPatchesStart() throws IOException, TinkerPatchException {
        newApkParser.parseResourceTable();
        final Map<String, ResourcePackage> newApkResPkgNameMap = newApkParser.getResourceTable().getPackageNameMap();
        do {
            if (newApkResPkgNameMap == null) {
                break;
            }

            final ResourcePackage newApkResPackage = newApkResPkgNameMap.get(newApkParser.getApkMeta().getPackageName());
            if (newApkResPackage == null) {
                break;
            }

            final Map<String, List<Type>> newApkResTypesNameMap = newApkResPackage.getTypesNameMap();
            if (newApkResTypesNameMap == null) {
                break;
            }

            final List<Type> newApkAnimResTypes = newApkResTypesNameMap.get("anim");
            if (newApkAnimResTypes == null) {
                break;
            }

            for (Type animType : newApkAnimResTypes) {
                for (ResourceEntry value : animType.getResourceEntryNameHashMap().values()) {
                    if (value == null) {
                        continue;
                    }
                    final ResourceValue resValue = value.getValue();
                    if (resValue == null) {
                        continue;
                    }
                    newApkAnimResNames.add(resValue.toStringValue());
                }
            }
        } while (false);
    }
```

**方法流程：**

* **步骤1：**对新包的进行解析，并取得与AndroidManifest.xml中包名匹配的package
* **步骤2：**获取anim类型的typeSpec，遍历typeSpec中的type，并用成员变量newApkAnimResNames把resValue中xml文件名称保存起来

**方法解读：**

* 这一步的主要原因是tinker不支持transition动画，所以会在生成补丁包之前把anim的xml文件保存起来，在后面进行warning提示

### ResDiffDecoder-> patch

```java
@Override
    public boolean patch(File oldFile, File newFile) throws IOException, TinkerPatchException {
        String name = getRelativePathStringToNewFile(newFile);

        //actually, it won't go below
        if (newFile == null || !newFile.exists()) {
            String relativeStringByOldDir = getRelativePathStringToOldFile(oldFile);
            if (Utils.checkFileInPattern(config.mResIgnoreChangePattern, relativeStringByOldDir)) {
                Logger.e("found delete resource: " + relativeStringByOldDir + " ,but it match ignore change pattern, just ignore!");
                return false;
            }
            deletedSet.add(relativeStringByOldDir);
            writeResLog(newFile, oldFile, TypedValue.DEL);
            return true;
        }

        File outputFile = getOutputPath(newFile).toFile();

        if (oldFile == null || !oldFile.exists()) {
            if (Utils.checkFileInPattern(config.mResIgnoreChangePattern, name)) {
                Logger.e("found add resource: " + name + " ,but it match ignore change pattern, just ignore!");
                return false;
            }
            FileOperation.copyFileUsingStream(newFile, outputFile);
            addedSet.add(name);
            writeResLog(newFile, oldFile, TypedValue.ADD);
            return true;
        }
        //both file length is 0
        if (oldFile.length() == 0 && newFile.length() == 0) {
            return false;
        }
        
        //new add file
        String newMd5 = MD5.getMD5(newFile);
        String oldMd5 = MD5.getMD5(oldFile);

        //oldFile or newFile may be 0b length
        if (oldMd5 != null && oldMd5.equals(newMd5)) {
            return false;
        }
        if (Utils.checkFileInPattern(config.mResIgnoreChangePattern, name)) {
            Logger.d("found modify resource: " + name + ", but it match ignore change pattern, just ignore!");
            return false;
        }
        if (name.equals(TypedValue.RES_MANIFEST)) {
            Logger.d("found modify resource: " + name + ", but it is AndroidManifest.xml, just ignore!");
            return false;
        }
        if (name.equals(TypedValue.RES_ARSC)) {
            if (AndroidParser.resourceTableLogicalChange(config)) {
                Logger.d("found modify resource: " + name + ", but it is logically the same as original new resources.arsc, just ignore!");
                return false;
            }
        }
        dealWithModifyFile(name, newMd5, oldFile, newFile, outputFile);
        return true;
    }

```

**方法流程：**

* 检查解压后的新旧apk中，oldFile对应的newFile不存在，则认为新的apk对oldFile资源删除，保存在deletedSet中，但是如注释所有，这一步的代码不会跑到，所以deleteSet依然为空的
* 检查解压后的新旧apk中，newFile对应的oldFile不存在，则认为新的apk新添了newFile资源，保存在addedSet中，并且把newFile拷贝到tinker_result目录下 
* 如果不存在上述两种情况，而且oldFile和newFile存在，对比oldFile和newFile的md5，相同则认为该资源文件没有做任何修改，直接返回 
* 如果该文件不在指定的ignoreChangePattern清单里，并且不是AndroidMainfest.xml文件，则：
  * 如果是resources.arsc文件，则开始进行新旧resources.arsc的对比
  * 如果不在ignoreChangePattern清单里，又不是AndroidManifest.xml文件，又不是resources.arsc文件，那就调用**dealWithModifyFile**执行处理

**解读：**

* 截止目前已经出现了2个set，addedSet和deleteSet
* addedSet在这一步进行了添加，储存了新增的资源文件
* deleteSet在这一步依然是空的，有可能在后面进行赋值
* AndroidManifest.xml的变化忽略是因为目前Tinker还不支持AndroidMainfest的热修复 ，并且在执行patch之前tinker就会对AndroidManifest.xml进行检查，通过检查AndroidManifest.xml的合法性避免导致为编译出的patch包带来风险

### ResDiffDecoder-> dealWithModifyFile 

```java
private boolean dealWithModifyFile(String name, String newMd5, File oldFile, File newFile, File outputFile) throws IOException {
        if (checkLargeModFile(newFile)) {
            if (!outputFile.getParentFile().exists()) {
                outputFile.getParentFile().mkdirs();
            }
            BSDiff.bsdiff(oldFile, newFile, outputFile);
            //treat it as normal modify
            if (Utils.checkBsDiffFileSize(outputFile, newFile)) {
                LargeModeInfo largeModeInfo = new LargeModeInfo();
                largeModeInfo.path = newFile;
                largeModeInfo.crc = FileOperation.getFileCrc32(newFile);
                largeModeInfo.md5 = newMd5;
                largeModifiedSet.add(name);
                largeModifiedMap.put(name, largeModeInfo);
                writeResLog(newFile, oldFile, TypedValue.LARGE_MOD);
                return true;
            }
        }
        modifiedSet.add(name);
        FileOperation.copyFileUsingStream(newFile, outputFile);
        writeResLog(newFile, oldFile, TypedValue.MOD);
        return false;
    }
```

**方法流程：**

* 判断新的文件是否为大文件(默认阀值为100kb)，如果是大文件则用BSDiff算法生成差量包，减少补丁包的大小
* 如果新的文件并不是大文件，那就直接把新文件添加到补丁包中，然后用modifiedSet记录下来

**解读：**

* 这里的优化处理就是，针对修改的文件，判断一下大小，到达阀值就差量下发，在下发成功合并，如果没到阀值，说明本来文件就不大，下发差量包的收益就下降了，这时候直接整个文件下发就可以，减少了合并时间
* 到这里已经出现了4个Set：addedSet、deleteSet(依然是空)、modifiedSet、largeModifiedSet

### ResDiffDecoder->  onAllPatchesEnd 

```java
 @Override
    public void onAllPatchesEnd() throws IOException, TinkerPatchException {
        //only there is only deleted set, we just ignore
        if (addedSet.isEmpty() && modifiedSet.isEmpty() && largeModifiedSet.isEmpty()) {
            return;
        }

        // ......
        
        //add delete set
        deletedSet.addAll(getDeletedResource(config.mTempUnzipOldDir, config.mTempUnzipNewDir));

        //we can't modify AndroidManifest file
        addedSet.remove(TypedValue.RES_MANIFEST);
        deletedSet.remove(TypedValue.RES_MANIFEST);
        modifiedSet.remove(TypedValue.RES_MANIFEST);
        largeModifiedSet.remove(TypedValue.RES_MANIFEST);
        //remove add, delete or modified if they are in ignore change pattern also
        removeIgnoreChangeFile(modifiedSet);
        removeIgnoreChangeFile(deletedSet);
        removeIgnoreChangeFile(addedSet);
        removeIgnoreChangeFile(largeModifiedSet);

        // after ignore-changes resource files are being removed, we now check if there's any anim
        // resources in added and modified files.
        checkIfSpecificResWasAnimRes(addedSet);
        checkIfSpecificResWasAnimRes(modifiedSet);
        checkIfSpecificResWasAnimRes(largeModifiedSet);

        // last add test res in assets for user cannot ignore it;
        addAssetsFileForTestResource();

        File tempResZip = new File(config.mOutFolder + File.separator + TEMP_RES_ZIP);
        final File tempResFiles = config.mTempResultDir;

        //gen zip resources_out.zip
        FileOperation.zipInputDir(tempResFiles, tempResZip, null);
        File extractToZip = new File(config.mOutFolder + File.separator + TypedValue.RES_OUT);

        String resZipMd5 = Utils.genResOutputFile(extractToZip, tempResZip, config,
            addedSet, modifiedSet, deletedSet, largeModifiedSet, largeModifiedMap);

        Logger.e("Final normal zip resource: %s, size=%d, md5=%s", extractToZip.getName(), extractToZip.length(), resZipMd5);
        logWriter.writeLineToInfoFile(
            String.format("Final normal zip resource: %s, size=%d, md5=%s", extractToZip.getName(), extractToZip.length(), resZipMd5)
        );
        //delete temp file
        FileOperation.deleteFile(tempResZip);

        //first, write resource meta first
        //use resources.arsc's base crc to identify base.apk
        String arscBaseCrc = FileOperation.getZipEntryCrc(config.mOldApkFile, TypedValue.RES_ARSC);
        String arscMd5 = FileOperation.getZipEntryMd5(extractToZip, TypedValue.RES_ARSC);
        if (arscBaseCrc == null || arscMd5 == null) {
            throw new TinkerPatchException("can't find resources.arsc's base crc or md5");
        }

        String resourceMeta = Utils.getResourceMeta(arscBaseCrc, arscMd5);
        writeMetaFile(resourceMeta);

        //pattern
        String patternMeta = TypedValue.PATTERN_TITLE;
        HashSet<String> patterns = new HashSet<>(config.mResRawPattern);
        //we will process them separate
        patterns.remove(TypedValue.RES_MANIFEST);

        writeMetaFile(patternMeta + patterns.size());
        //write pattern
        for (String item : patterns) {
            writeMetaFile(item);
        }

        //add store files
        getCompressMethodFromApk();

        //write meta file, write large modify first
        writeMetaFile(largeModifiedSet, TypedValue.LARGE_MOD);
        writeMetaFile(modifiedSet, TypedValue.MOD);
        writeMetaFile(addedSet, TypedValue.ADD);
        writeMetaFile(deletedSet, TypedValue.DEL);
        writeMetaFile(storedSet, TypedValue.STORED);

    }
```

**方法流程：**

* 对addedSet、modifiedSet、largeModifiedSet进行空校验，如果都是空就说明不需要处理
* 根据旧新的解压目录获取被删除的文件并填入deletedSet中，这里patch不一样，是以旧包为遍历，所以不会在出现不执行的问题
* 移除addedSet、modifiedSet、largeModifiedSet、deletedSet中的AndroidManifest.xml以及ignore文件
* 检查在addedSet、modifiedSet、largeModifiedSet中是否有anim类型的资源
* 添加一个测试资源文件
* 构建一个resources_out.zip，并生成一个meta file，用于补丁下发时候的校验，meta file的组成为：
  * resources_out.zip+旧包arsc文件crc+resources_out.zip的md5
  * pattern：+数目
  * pattern如：resources.arsc、res/* 、asset/*
  * 依次写入largeModifiedSet、modifiedSet、addedSet、deletedSet、storedSet的标题与数目

### 小结

到这里Tinker资源热修复的资源补丁包生成流程就结束，小结一下流程及优化设计：

* 遍历新包，然后根据文件名与旧包组成对应文件的旧版地址，然后把新增的资源放入addedSet中
* 根据md5判断两个文件是否发生改变，这里有优化处理就是如果文件大小到达一定阀值(默认100kb)就用BSDiff生成常量文件进行下发，否则就全文件下发
* 遍历旧包，然后一样的判断方式，把移除的资源放入deletedSet中
* 生成meta file用于记录一些关键信息，用于补丁下发成功后的合并

下一阶段的源码解析就是针对Tinker对资源补丁包下发成功后合并处理了

### ResDiffPatchInternal-> tryRecoverResourceFiles 

```java
protected static boolean tryRecoverResourceFiles(Tinker manager, ShareSecurityCheck checker, Context context,
                                                String patchVersionDirectory, File patchFile) {

        if (!manager.isEnabledForResource()) {
            TinkerLog.w(TAG, "patch recover, resource is not enabled");
            return true;
        }
        String resourceMeta = checker.getMetaContentMap().get(RES_META_FILE);

        if (resourceMeta == null || resourceMeta.length() == 0) {
            TinkerLog.w(TAG, "patch recover, resource is not contained");
            return true;
        }

        long begin = SystemClock.elapsedRealtime();
        boolean result = patchResourceExtractViaResourceDiff(context, patchVersionDirectory, resourceMeta, patchFile);
        long cost = SystemClock.elapsedRealtime() - begin;
        TinkerLog.i(TAG, "recover resource result:%b, cost:%d", result, cost);
        return result;
    }
```

**方法流程：**

* 判断是否支持资源热修复
* 判断补丁中的resourceMeta也就是补丁包生成的最后一步meta file的内容是否存在
* 计时并最转调paycajResourceExtractViaResourceDiff方法

### ResDiffPatchInternal-> patchResourceExtractViaResourceDiff 

```java
private static boolean patchResourceExtractViaResourceDiff(Context context, String                                                                patchVersionDirectory,String meta, File patchFile) {
        String dir = patchVersionDirectory + "/" + ShareConstants.RES_PATH + "/";

        if (!extractResourceDiffInternals(context, dir, meta, patchFile, TYPE_RESOURCE)) {
            TinkerLog.w(TAG, "patch recover, extractDiffInternals fail");
            return false;
        }
        return true;
    }
```

### ResDiffPatchInternal-> extractResourceDiffInternals

```java
private static boolean extractResourceDiffInternals(Context context, String dir, String meta, File patchFile, int type) {
       
        ShareResPatchInfo resPatchInfo = new ShareResPatchInfo();
        ShareResPatchInfo.parseAllResPatchInfo(meta, resPatchInfo);

        if (!SharePatchFileUtil.checkIfMd5Valid(resPatchInfo.resArscMd5)) {
            TinkerLog.w(TAG, "resource meta file md5 mismatch, type:%s, md5: %s", ShareTinkerInternals.getTypeString(type), resPatchInfo.resArscMd5);
            manager.getPatchReporter().onPatchPackageCheckFail(patchFile, BasePatchInternal.getMetaCorruptedCode(type));
            return false;
        }
    
        File directory = new File(dir);
        File tempResFileDirectory = new File(directory, "res_temp");
        File resOutput = new File(directory, ShareConstants.RES_NAME);
        //check result file whether already exist
        if (resOutput.exists()) {
            if (SharePatchFileUtil.checkResourceArscMd5(resOutput, resPatchInfo.resArscMd5)) {
                //it is ok, just continue
                TinkerLog.w(TAG, "resource file %s is already exist, and md5 match, just return true", resOutput.getPath());
                return true;
            } else {
                TinkerLog.w(TAG, "have a mismatch corrupted resource " + resOutput.getPath());
                resOutput.delete();
            }
        } else {
            resOutput.getParentFile().mkdirs();
        }

        try {
            ApplicationInfo applicationInfo = context.getApplicationInfo();
            if (applicationInfo == null) {
                //Looks like running on a test Context, so just return without patching.
                TinkerLog.w(TAG, "applicationInfo == null!!!!");
                return false;
            }
            
            
            String apkPath = applicationInfo.sourceDir;
            if (!checkAndExtractResourceLargeFile(context, apkPath, directory, tempResFileDirectory, patchFile, resPatchInfo, type)) {
                return false;
            }

            TinkerZipOutputStream out = null;
            TinkerZipFile oldApk = null;
            TinkerZipFile newApk = null;
            int totalEntryCount = 0;
            try {
                out = new TinkerZipOutputStream(new BufferedOutputStream(new FileOutputStream(resOutput)));
                oldApk = new TinkerZipFile(apkPath);
                newApk = new TinkerZipFile(patchFile);
                final Enumeration<? extends TinkerZipEntry> entries = oldApk.entries();
                while (entries.hasMoreElements()) {
                    TinkerZipEntry zipEntry = entries.nextElement();
                    if (zipEntry == null) {
                        throw new TinkerRuntimeException("zipEntry is null when get from oldApk");
                    }
                    String name = zipEntry.getName();
                    if (name.contains("../")) {
                        continue;
                    }
                    if (ShareResPatchInfo.checkFileInPattern(resPatchInfo.patterns, name)) {
                        //won't contain in add set.
                        if (!resPatchInfo.deleteRes.contains(name)
                            && !resPatchInfo.modRes.contains(name)
                            && !resPatchInfo.largeModRes.contains(name)
                            && !name.equals(ShareConstants.RES_MANIFEST)) {
                            TinkerZipUtil.extractTinkerEntry(oldApk, zipEntry, out);
                            totalEntryCount++;
                        }
                    }
                }

                //process manifest
                TinkerZipEntry manifestZipEntry = oldApk.getEntry(ShareConstants.RES_MANIFEST);
                if (manifestZipEntry == null) {
                    TinkerLog.w(TAG, "manifest patch entry is null. path:" + ShareConstants.RES_MANIFEST);
                    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, ShareConstants.RES_MANIFEST, type);
                    return false;
                }
                TinkerZipUtil.extractTinkerEntry(oldApk, manifestZipEntry, out);
                totalEntryCount++;

                for (String name : resPatchInfo.largeModRes) {
                    TinkerZipEntry largeZipEntry = oldApk.getEntry(name);
                    if (largeZipEntry == null) {
                        TinkerLog.w(TAG, "large patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    ShareResPatchInfo.LargeModeInfo largeModeInfo = resPatchInfo.largeModMap.get(name);
                    TinkerZipUtil.extractLargeModifyFile(largeZipEntry, largeModeInfo.file, largeModeInfo.crc, out);
                    totalEntryCount++;
                }

                for (String name : resPatchInfo.addRes) {
                    TinkerZipEntry addZipEntry = newApk.getEntry(name);
                    if (addZipEntry == null) {
                        TinkerLog.w(TAG, "add patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    if (resPatchInfo.storeRes.containsKey(name)) {
                        File storeFile = resPatchInfo.storeRes.get(name);
                        TinkerZipUtil.extractLargeModifyFile(addZipEntry, storeFile, addZipEntry.getCrc(), out);
                    } else {
                        TinkerZipUtil.extractTinkerEntry(newApk, addZipEntry, out);
                    }
                    totalEntryCount++;
                }

                for (String name : resPatchInfo.modRes) {
                    TinkerZipEntry modZipEntry = newApk.getEntry(name);
                    if (modZipEntry == null) {
                        TinkerLog.w(TAG, "mod patch entry is null. path:" + name);
                        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type);
                        return false;
                    }
                    if (resPatchInfo.storeRes.containsKey(name)) {
                        File storeFile = resPatchInfo.storeRes.get(name);
                        TinkerZipUtil.extractLargeModifyFile(modZipEntry, storeFile, modZipEntry.getCrc(), out);
                    } else {
                        TinkerZipUtil.extractTinkerEntry(newApk, modZipEntry, out);
                    }
                    totalEntryCount++;
                }
                // set comment back
                out.setComment(oldApk.getComment());
            } finally {
                IOHelper.closeQuietly(out);
                IOHelper.closeQuietly(oldApk);
                IOHelper.closeQuietly(newApk);

                //delete temp files
                SharePatchFileUtil.deleteDir(tempResFileDirectory);
            }
            boolean result = SharePatchFileUtil.checkResourceArscMd5(resOutput, resPatchInfo.resArscMd5);

            if (!result) {
                TinkerLog.i(TAG, "check final new resource file fail path:%s, entry count:%d, size:%d", resOutput.getAbsolutePath(), totalEntryCount, resOutput.length());
                SharePatchFileUtil.safeDeleteFile(resOutput);
                manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, ShareConstants.RES_NAME, type);
                return false;
            }

            TinkerLog.i(TAG, "final new resource file:%s, entry count:%d, size:%d", resOutput.getAbsolutePath(), totalEntryCount, resOutput.length());
        } catch (Throwable e) {
            throw new TinkerRuntimeException("patch " + ShareTinkerInternals.getTypeString(type) +  " extract failed (" + e.getMessage() + ").", e);
        }
        return true;
    }
```

**方法流程：**

* 读取meta file文件并保存在ShareResPatchInfo中，然后根据md5的值进行有效校验
* 创建一个res_temp临时文件夹
* 创建resources.apk用于承载合并的结果，然后判断本地是否有这个文件存在，有的话就进行md5匹配，匹配成功就说明补丁合并完成了，匹配不成功就直接把本地文件删除
* 然后便调用checkAndExtractResourceLargeFile对arsc文件的crc进行校验并用BSDiff算法合并大文件，然后把合并的大文件存储在largeModeInfo的file中，然后就把meta file中的store文件转存到res_temp
* 最后便根据旧包、补丁包生成新包：
  * 如果文件存在pattern中而且不存在deletedSet、addedSet、modifySet、largeModifySet并且不是manifest的话就直接迁移就好，当然resources.arsc也包含在这个pattern里
  * 拷贝manifest到resources.apk中，当然这个manifest是不能变
  * 把大文件写入到resources.apk中，而且大文件写入采用的是store归档的方式，感觉应该是想系统在读取大文件的时候可以直接mmap把
  * 把addedSet中新增的资源写入到resources.apk中，实际采用存储方式根据文件自身的存储方式来决定，判断依据就是是否在storeSet中
  * 把modifyedSet中新增的资源写入到resources.apk中，实际采用存储方式根据文件自身的存储方式来决定，判断依据就是是否在storeSet中
  * 把旧apk的comment字段迁移到resources.apk中

**解读：**

* 在补丁合并过程中起关键作用的除了resources_out.zip之外还有meta file文件，这个文件处理储存一些crc、md5等的校验信息之外，还储存了deletedSet、addedSet、modifySet、largeModifySet的序列化结果
* 在补丁合并过程中一个小细节就是存储方式，对大文件进行归档存储，对其他文件就采用默认存储方式
* 而可以看到其实合并补丁的过程中是会忽略一些删除的文件，而序列化deletedSet的目的其实就是移除pattern中被删除的文件

### TinkerResourcesLoader-> loadTinkerResources

```java
public static boolean loadTinkerResources(TinkerApplication application, String directory, Intent intentResult) {
        if (resPatchInfo == null || resPatchInfo.resArscMd5 == null) {
            return true;
        }
        String resourceString = directory + "/" + RESOURCE_PATH +  "/" + RESOURCE_FILE;
        File resourceFile = new File(resourceString);
        long start = System.currentTimeMillis();
        
        // ......
        try {
            TinkerResourcePatcher.monkeyPatchExistingResources(application, resourceString);
            Log.i(TAG, "monkeyPatchExistingResources resource file:" + resourceString + ", use time: " + (System.currentTimeMillis() - start));
        } catch (Throwable e) {
            // ......
            return false;
        }
        // tinker resources loaded, monitor runtime accident
        ResourceStateMonitor.tryStart(application);
        return true;
    }
```

**方法流程：**

* 根据提前定义的文件目录和文件名称拿到合并后的resources.apk
* 然后转调monkeyPatchExistingResources方法执行加载

### TinkerResourcePatcher-> monkeyPatchExistingResources

```java
public static void monkeyPatchExistingResources(Context context, String externalResourceFile) throws Throwable {
        if (externalResourceFile == null) {
            return;
        }

        final ApplicationInfo appInfo = context.getApplicationInfo();

        final Field[] packagesFields;
        if (Build.VERSION.SDK_INT < 27) {
            packagesFields = new Field[]{packagesFiled, resourcePackagesFiled};
        } else {
            packagesFields = new Field[]{packagesFiled};
        }
        for (Field field : packagesFields) {
            final Object value = field.get(currentActivityThread);

            for (Map.Entry<String, WeakReference<?>> entry
                    : ((Map<String, WeakReference<?>>) value).entrySet()) {
                final Object loadedApk = entry.getValue().get();
                if (loadedApk == null) {
                    continue;
                }
                final String resDirPath = (String) resDir.get(loadedApk);
                if (appInfo.sourceDir.equals(resDirPath)) {
                    resDir.set(loadedApk, externalResourceFile);
                }
            }
        }

        // Create a new AssetManager instance and point it to the resources installed under
        if (((Integer) addAssetPathMethod.invoke(newAssetManager, externalResourceFile)) == 0) {
            throw new IllegalStateException("Could not create new AssetManager");
        }

        // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
        // in L, so we do it unconditionally.
        if (stringBlocksField != null && ensureStringBlocksMethod != null) {
            stringBlocksField.set(newAssetManager, null);
            ensureStringBlocksMethod.invoke(newAssetManager);
        }

        for (WeakReference<Resources> wr : references) {
            final Resources resources = wr.get();
            if (resources == null) {
                continue;
            }
            // Set the AssetManager of the Resources instance to our brand new one
            try {
                //pre-N
                assetsFiled.set(resources, newAssetManager);
            } catch (Throwable ignore) {
                // N
                final Object resourceImpl = resourcesImplFiled.get(resources);
                // for Huawei HwResourcesImpl
                final Field implAssets = findField(resourceImpl, "mAssets");
                implAssets.set(resourceImpl, newAssetManager);
            }

            clearPreloadTypedArrayIssue(resources);

            resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
        }

        // Handle issues caused by WebView on Android N.
        // Issue: On Android N, if an activity contains a webview, when screen rotates
        // our resource patch may lost effects.
        // for 5.x/6.x, we found Couldn't expand RemoteView for StatusBarNotification Exception
        if (Build.VERSION.SDK_INT >= 24) {
            try {
                if (publicSourceDirField != null) {
                    publicSourceDirField.set(context.getApplicationInfo(), externalResourceFile);
                }
            } catch (Throwable ignore) {
                // Ignored.
            }
        }

        if (!checkResUpdate(context)) {
            throw new TinkerRuntimeException(ShareConstants.CHECK_RES_INSTALL_FAIL);
        }
    }
```

**方法流程：**

* 对Android8.1以下版本的兼容操作：取出ActivityThread中的mPackages和mResourcePackages，这两个集合是在ActivityThread中用于缓存LoadedApk的，而这一步实际的任务是要修改LoadedApk中的mResDir，也就是源apk，把它替换成热修复合并的resources.apk
* 反射调用一下ensureStringBlocks这个方法，主要目的就是兼容Android低版本，因为在低版本中这个方法会从native层中拿到对应的StringBlock然后存在应用层中，但是在Android高版本，这个方法就直接return null，因为在ApkAssets的创建中就会直接去那native层的全局常量池StringBlock，所以这也是注释说的调用不会出问题的原因把
* 最后其实和instant run一致，创建新的AssetManager，然后把引用的地方作修改

### 总结

其实Tinker资源热修复中加载这部分和instant run相差不大，主要的特点就是生成差量包，然后下发差量包，最后在本地进行合并生成resources.apk

但是读完Tinker源码依然有一个问题没有解决：

* 对于deletedSet，为什么不在补丁包的生成就直接从pattern中移除，而要等到合并的时候在做呢？

希望等我技术成长的一天，也能解决这些我今天解决不了的问题

## Sophix 

Sophix的原理就是生成一个packge id为0x66的资源包，然后不与0x7f冲突，因此直接加入到已有的AssetManager中就可以直接使用了，而下发的资源包里质只包含原包里没有而新包里有的资源，以及内容发生变化的资源

对于资源改变的三种情况，处理为：

* 对于新增资源，直接加入补丁包，然后新代码里直接引用就可以了、
* 对于减少资源，只需要不使用它就行了，因此不需要考虑这种情况
* 对于修改资源，把它视为新增资源，在打入补丁的时候，引用处的旧id改为新id

Q:为什么要生成一个package id为0x66的资源包呢？用0x7f不香吗？

A1:原因之1是为了兼容Android版本的变化，因为在Android低版本，调用AssetManager的addAssetPath方法只会把资源包添加到mAssetPath中，并不会执行真正的解析，而执行解析操作的时候其实是在第一次读取的时候，而且即便我们之前没做过任何资源相关的操作，Android framework里的代码也会多次调到那里，但是因为读取过后会有缓存，所以所以低版本中补丁包里的资源是完全不生效的

A2:原因之2还是Android版本的变化，因为在不同的系统版本中对PackageGroup的遍历方向也是会发生改变的，有的是从前往后，有的是从后往前，这样其实兼容起来不是那么好做，并且在多次遍历的情况下甚至可能会出现entry丢失的情况！

A3:原因之3还是在于即便在同一个pakcgae group中仍有可能由位遍历顺序而无法访问到相同config的资源，所以导致补丁的失效

这么多问题，所以Android官方才会使用新建AssetManager的方式去做资源热秀，而有官方的撑腰，这也是这套方案那么多大厂在用的原因了吧

这套方案针对Android4.4以下的版本会销毁java层的AssetManager中native层的AssetManager，然后重建一个，用这种方式实现优雅化的替换

这套方案的难点也是在于，需要对resources.arsc文件进行比对，然后生成一个差量resources.arsc文件

但是其实有点好奇，如果对一个已经修复过的包再一次进行热修复会怎么处理，会生成一个另外的packageId？还是会做合并呢？不然的话不就又有了packageId 0x66重复的问题了吗？





参考资料：

[微信Tinker资源热修复解析](<https://hellokugo.github.io/2016/11/30/%E5%BE%AE%E4%BF%A1Tinker%E8%B5%84%E6%BA%90%E7%83%AD%E4%BF%AE%E5%A4%8D%E8%A7%A3%E6%9E%90/>)

[热修复中的资源修复](<http://zjutkz.net/2016/07/22/%E7%83%AD%E4%BF%AE%E5%A4%8D%E4%B8%AD%E7%9A%84%E8%B5%84%E6%BA%90%E4%BF%AE%E5%A4%8D/>)

[Android插件化与热修复(七)-微信Tinker资源加载、gradle插件分析](<https://www.jianshu.com/p/3fd9789751bc>)

《深入探索Android热修复技术原理》