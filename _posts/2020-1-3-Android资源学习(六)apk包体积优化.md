---
layout:     post   				    
title:      Android资源学习(六)apk包体积优化
subtitle:   Android资源学习系列   #副标题
date:       2020-1-3		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-keybord.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Android资源学习系列
---

# Android资源学习(六)apk包体积优化

## 资源混淆

### aapt

通过修改aapt在生成resources.arsc和*ap_时把资源文件的名称进行替换

```java
tatic status_t makeFileResources(Bundle* bundle, const sp<AaptAssets>& assets,
                                      ResourceTable* table,
                                      const sp<ResourceTypeSet>& set,
                                      const char* resType)
    {
        String8 type8(resType);
        String16 type16(resType);

        bool hasErrors = false;

        ResourceDirIterator it(set, String8(resType));
        ssize_t res;
        while ((res=it.next()) == NO_ERROR) {
            if (bundle->getVerbose()) {
                printf("    (new resource id %s from %s)\n",
                       it.getBaseName().string(), it.getFile()->getPrintableSource().string());
            }
            String16 baseName(it.getBaseName());
            const char16_t* str = baseName.string();
            const char16_t* const end = str + baseName.size();
            while (str < end) {
                if (!((*str >= 'a' && *str <= 'z')
                        || (*str >= '0' && *str <= '9')
                        || *str == '_' || *str == '.')) {
                    fprintf(stderr, "%s: Invalid file name: must contain only [a-z0-9_.]\n",
                            it.getPath().string());
                    hasErrors = true;
                }
                str++;
            }
            String8 resPath = it.getPath();
            resPath.convertToResPath();
            
            String8 obfuscationName;
            String8 obfuscationPath = getObfuscationName(resPath, obfuscationName);
            
            table->addEntry(SourcePos(it.getPath(), 0), String16(assets->getPackage()),
                            type16,
                            baseName, // String16(obfuscationName),
                            String16(obfuscationPath), // resPath
                            NULL,
                            &it.getParams());
            assets->addResource(it.getLeafName(), obfuscationPath/*resPath*/, it.getFile(), type8);
        }

        return hasErrors ? UNKNOWN_ERROR : NO_ERROR;
    }
```

在ResourceTable和Assets中添加资源文件时， 对资源文件名称进行修改，这就能够做到资源文件名称的替换，这样通过使用修改过的AAPT编译资源并进行打包 

### AndResGuard

mark下之前在做源码解析时的笔记，点开就可以查看当时的笔记链接嘿嘿

[微信资源混淆AndResGuard源码解析(一)](<https://cc1over.github.io/2019/08/08/%E5%BE%AE%E4%BF%A1%E8%B5%84%E6%BA%90%E6%B7%B7%E6%B7%86AndResGuard%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/> )

[微信资源混淆AndResGuard源码解析(二)](<https://cc1over.github.io/2019/08/09/%E5%BE%AE%E4%BF%A1%E8%B5%84%E6%BA%90%E6%B7%B7%E6%B7%86AndResGuard%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(%E4%BA%8C)/>)

[微信资源混淆AndResGuard源码解析(三)](<https://cc1over.github.io/2019/08/10/%E5%BE%AE%E4%BF%A1%E8%B5%84%E6%BA%90%E6%B7%B7%E6%B7%86AndResGuard%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(%E4%B8%89)/> )

## 移除无用资源

### RemoveUnusedResourcesTask-> removeUnusedResources

```groovy
void removeUnusedResources(String originalApk, String rTxtFile, SigningConfig signingConfig) {
    ZipOutputStream zipOutputStream = null;
    boolean needSign = project.extensions.matrix.removeUnusedResources.needSign
    boolean shrinkArsc = project.extensions.matrix.removeUnusedResources.shrinkArsc
    String apksigner = project.extensions.matrix.removeUnusedResources.apksignerPath
    if(needSign) {
       // 签名信息和签名工具有效性校验  
    }  
    try {
        File inputFile = new File(originalApk);
        Set<String> ignoreRes = project.extensions.matrix.removeUnusedResources.ignoreResources;
        for (String res : ignoreRes) {
           ignoreResources.add(Util.globToRegexp(res));
        }
        Set<String> unusedResources = project.extensions.matrix.removeUnusedResources.unusedResources;
                Iterator<String> iterator = unusedResources.iterator();
         Set<String> unusedResources = project.extensions.matrix.removeUnusedResources.unusedResources;
         Iterator<String> iterator = unusedResources.iterator();
         String res = null;
         while (iterator.hasNext()) {
               res = iterator.next();
               if (ignoreResource(res)) {
                   iterator.remove();
                   Log.i(TAG, "ignore unused resources %s", res);
               }
         } 
         Log.i(TAG, "unused resources count:%d", unusedResources.size());

         String outputApk = inputFile.getParentFile().getAbsolutePath() + "/" + inputFile.getName().substring(0, inputFile.getName().indexOf('.')) + "_shrinked.apk";

         File outputFile = new File(outputApk);
         if (outputFile.exists()) {
            Log.w(TAG, "output apk file %s is already exists! It will be deleted anyway!", outputApk);
            outputFile.delete();
            outputFile.createNewFile();
        }
        
        ZipFile zipInputFile = new ZipFile(inputFile);
        zipOutputStream = new ZipOutputStream(new FileOutputStream(outputFile));
        // ......
    } finally {
      if (zipOutputStream != null) {
         zipOutputStream.close()
      }  
    }
    // ......
}
```

这个方法比较长，笔者把它拆开成几个部分来看

第一部分主要是做移除工作的预备工作

通过extension把参数导入进来：

* boolean needSign
* boolean shrinkArsc
* String apksignerPath
* Set<String> unusedResources
* Set<String> ignoreResources

needSign和apksignerPath是用于判断是否能正常进行移除资源后的签名操作，不能就直接抛出异常

然后会传进来两个Set，分别是unusedResources和ignoreResources，代表无用资源的集合以及忽略的资源集合，然后便是从unusedResources中移除那些忽略的部分

所以这个Task并不会去找那些没有用到的资源，而是由外界传进来，然后Task只负责移除，提供了一定的动态性

然后构建一个shrinked.apk文件，然后做本地检查

### RemoveUnusedResourcesTask-> removeUnusedResources

```groovy
void removeUnusedResources(String originalApk, String rTxtFile, SigningConfig signingConfig) {
    // ......
    Map<String, Integer> resourceMap = new HashMap();
    Map<String, Pair<String, Integer>[]> styleableMap = new HashMap();
    File resTxtFile = new File(rTxtFile);
    readResourceTxtFile(resTxtFile, resourceMap, styleableMap);

    Map<String, Integer> removeResources = new HashMap<>();
    for (String resName : unusedResources) {
       if (!ignoreResource(resName)) {
           removeResources.put(resName, resourceMap.remove(resName));
        }
    }   
    
   // ...... 
}
```

第二部分主要是做R.txt的解析以及数据结构的初始化

创建了resourceMap，针对R.txt中一般Res 如：string abc_action_bar_home_description 0x7f090000 

创建了styleableMap，针对R.txt中styleable Res 如：int[] styleable TagLayout { 0x010100af, 0x7f0102b5, 0x7f0102b6 }           或者 int styleable TagLayout_android_gravity 0 

然后只需要解析R.txt然后把内容填充到这两个map中就可以了

然后在resourceMap中过滤掉那些忽略的资源，转存到removeResources这个集合中

### RemoveUnusedResourcesTask-> removeUnusedResources

```groovy
void removeUnusedResources(String originalApk, String rTxtFile, SigningConfig signingConfig) {
    // ......
    for (ZipEntry zipEntry : zipInputFile.entries()) {
                    if (zipEntry.name.startsWith("res/")) {
                        String resourceName = entryToResouceName(zipEntry.name);
                        if (!Util.isNullOrNil(resourceName)) {
                            if (removeResources.containsKey(resourceName)) {
                                Log.i(TAG, "remove unused resource %s", resourceName);
                                continue;
                            } else {
                                addZipEntry(zipOutputStream, zipEntry, zipInputFile);
                            }
                        } else {
                            addZipEntry(zipOutputStream, zipEntry, zipInputFile);
                        }
                    } else {
                        if (needSign && zipEntry.name.startsWith("META-INF/")) {
                            continue;
                        } else {
                            if (shrinkArsc && zipEntry.name.equalsIgnoreCase("resources.arsc") && unusedResources.size() > 0) {
                                File srcArscFile = new File(inputFile.getParentFile().getAbsolutePath() + "/resources.arsc");
                                File destArscFile = new File(inputFile.getParentFile().getAbsolutePath() + "/resources_shrinked.arsc");
                                if (srcArscFile.exists()) {
                                    srcArscFile.delete();
                                    srcArscFile.createNewFile();
                                }
                                unzipEntry(zipInputFile, zipEntry, srcArscFile);

                                ArscReader reader = new ArscReader(srcArscFile.getAbsolutePath());
                                ResTable resTable = reader.readResourceTable();
                                for (String resName : removeResources.keySet()) {
                                    ArscUtil.removeResource(resTable, removeResources.get(resName), resName);
                                }
                                ArscWriter writer = new ArscWriter(destArscFile.getAbsolutePath());
                                writer.writeResTable(resTable);
                                Log.i(TAG, "shrink resources.arsc size %f KB", (srcArscFile.length() - destArscFile.length()) / 1024.0);
                                addZipEntry(zipOutputStream, zipEntry, destArscFile);
                            } else {
                                addZipEntry(zipOutputStream, zipEntry, zipInputFile);
                            }
                        }
                    }
                }
    // ......
}
```

第三部分就是做实质的input.apk到_shrinked.apk的迁移以及resources.arsc文件的处理

这一步会遍历apk中的entry，然后先做名称转换，把entry name转换成R.xxx.xxxx的形式，目的就是为了和extension传进来的形式对应起来，然后对文件的处理比较简单，如果在map中，那就不写到_shrinked.apk中，如果不在，那就可以写了

**然后关键的resources.arsc文件的处理来了：**

* 用ArscReader把源resources.arsc文件读进来生成ResTable
* 用ArscUtil的removeResource方法去在ResTable中移除
* 最后用ArscWriter向目的resources.arsc文件把ResTable写进去

所以这位作者他的设计是用序列化和反序列化的方式实现resources.arsc文件的处理，把这个文件读到内存中构建一种数据结构，然后再内存中修改后再序列化回去生成一个新的问题，十分厉害！

这种做法难度会比较高，其实最开始我的想法只是修改aapt的源码然后在收集的时候剔除名单里的资源，然后再走一次编译过程，但是像Matrix这样做确实更好，不需要干涉编译的过程，而且维护还很方便

### ArscUtil-> removeResource 

```java
public static void removeResource(ResTable resTable, int resourceId, String resourceName) throws IOException {
        ResPackage resPackage = findResPackage(resTable, getPackageId(resourceId));
        if (resPackage != null) {
            List<ResType> resTypeList = findResType(resPackage, resourceId);
            for (ResType resType : resTypeList) {
                int entryId = getResourceEntryId(resourceId);
                Log.i(TAG, "try to remove %s (%H), find resource %s", resourceName, resourceId, ArscUtil.resolveStringPoolEntry(resPackage.getResNamePool().getStrings().get(resType.getEntryTable().get(entryId).getStringPoolIndex()).array(), resPackage.getResNamePool().getCharSet()));
                resType.getEntryTable().set(entryId, null);
                resType.getEntryOffsets().set(entryId, ArscConstants.NO_ENTRY_INDEX);
                resType.refresh();
            }
            resPackage.refresh();
            resTable.refresh();
        }
    }
```

把resourceId解析成packageId，然后从ResTable中拿到指定的ResPackage

把resourceId解析成typeId，然后从ResPackage中拿到resType的集合

调用reolveStringPoolEntry方法把这个资源对应的资源名称列表拿到并且log出来

然后把resType中对应entry置为null，并且给予offset数组0xFFFF值标记这里没有索引

然后就是从内到外执行一次refresh操作

### ArscUtil-> findResPackage

```java
public static ResPackage findResPackage(ResTable resTable, int packageId) {
        ResPackage resPackage = null;
        for (ResPackage pkg : resTable.getPackages()) {
            if (pkg.getId() == packageId) {
                resPackage = pkg;
                break;
            }
        }
        return resPackage;
    }
```

### ArscUtil-> findResType

```java
public static List<ResType> findResType(ResPackage resPacakge, int resourceId) {
        ResType resType = null;
        int typeId = (resourceId & 0X00FF0000) >> 16;
        int entryId = resourceId & 0x0000FFFF;
        List<ResType> resTypeList = new ArrayList<ResType>();
        List<ResChunk> resTypeArray = resPacakge.getResTypeArray();
        if (resTypeArray != null) {
            for (int i = 0; i < resTypeArray.size(); i++) {
                if (resTypeArray.get(i).getType() == ArscConstants.RES_TABLE_TYPE_TYPE
                        && ((ResType) resTypeArray.get(i)).getId() == typeId) {
                    int entryCount = ((ResType) resTypeArray.get(i)).getEntryCount();
                    if (entryId < entryCount) {
                        int offset = ((ResType) resTypeArray.get(i)).getEntryOffsets().get(entryId);
                        if (offset != ArscConstants.NO_ENTRY_INDEX) {
                            resType = ((ResType) resTypeArray.get(i));
                            resTypeList.add(resType);
                        }
                    }
                }
            }
        }
        return resTypeList;
    }
```

先从ResPackage中拿到一个typeArray，因为在package抽象出来的数据结构中是直接用ResChunk抽象typeSpec和type的，然后拿出来之后再转型

然后根据typeId匹配到ResType之后，然后就可以根据最后拆出来的entryId来查看ResType中是否有对应的ResEntry，有就把它添加到结果集中

这里会把所有ResType都返回了，毕竟这个时候没有Config能匹配，而且删除无用资源也肯定是要把所有ResType中都删掉的

### ResType-> fresh

```java
public void refresh() throws IOException {
        //校正entryOffsets
        int lastOffset = 0;
        for (int i = 0; i < entryCount; i++) {
            if (entryOffsets.get(i) != ArscConstants.NO_ENTRY_INDEX) {
                entryOffsets.set(i, lastOffset);
                lastOffset += entryTable.get(i).toBytes().length;
            }
        }
        recomputeChunkSize();
 }
```

结合fresh函数，回顾在removeResource方法中对ResType的处理：

```java
resType.getEntryTable().set(entryId, null);
resType.getEntryOffsets().set(entryId, ArscConstants.NO_ENTRY_INDEX);
resType.refresh();
```

结合这两段代码就知道是如何keep住之前的id的同时又把无用的删去，做法就是：

* 对于无用资源，只把entry给它删掉，保留下来的entry的都是有用的
* 但是offset数组的数量和entryCount都不去动它
* 然后把通过重新设置偏移量就可以修正这个offset所指向的entry了

![](https://raw.githubusercontent.com/Cc1over/Cc1over.github.io/master/img/matrix-unuse-resources.png)

### 总结

所以其实Matrix中的RemoveUnusedResourcesTask的处理分两步走：

* 做一个新的apk，然后把原来apk的文件拷贝过去，其中把无用的资源文件过滤掉
* 把resources.arsc文件中与无用资源相关的entry删掉，然后给offset重定向

但是其实这套RemoveUnusedResources的方案并没有对常量池的内容进行修改，它把与无用资源相关的entry删掉了，但是为什么不顺便把常量池的内容也删掉呢？如果对常量池的修改用类似的偏移量处理不是也可以做到吗，为什么会忽略这一步呢？难道是有什么坑是未知的吗！？还是说是作者觉得这部分占的空间不多，就让它留着吧？

希望技术成长的以后能解决我今天的疑问！







参考资料：

[美团Android资源混淆保护实践](<https://tech.meituan.com/2015/09/30/mt-android-resource-obfuscation.html>)