---
layout:     post   				    
title:      微信资源混淆AndResGuard源码解析(三) 
subtitle:   apk学习系列   #副标题
date:       2019-8-10		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-map.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - apk学习系列
---

# 微信资源混淆AndResGuard源码解析(三)

## 前言

前两篇文章主要记录了，resources.arsc的读和写的流程和细节，这一篇文章，记录的主要就是在resources.arsc写回操作完成之后，构建apk的流程和细节

## [Main-> buildApk]

```java
 private void buildApk(
      ApkDecoder decoder, File apkFile, File outputFile, InputParam.SignatureType signatureType, int minSDKVersion)
      throws Exception {
    ResourceApkBuilder builder = new ResourceApkBuilder(config);
    String apkBasename = apkFile.getName();
    apkBasename = apkBasename.substring(0, apkBasename.indexOf(".apk"));
    builder.setOutDir(mOutDir, apkBasename, outputFile);
    switch (signatureType) {
      case SchemaV1:
        builder.buildApkWithV1sign(decoder.getCompressData());
        break;
      case SchemaV2:
        builder.buildApkWithV2sign(decoder.getCompressData(), minSDKVersion);
        break;
    }
  }
```

* 创建一个**ResourceApkBuilder**对象
* 然后分别根据V1和V2签名，用**ResourceApkBuilder**构建出apk，特别关注的就是decoder.getCompressData()，这里包含了一些压缩处理是之前还没有太多留意的

## [ResourceApkBuilder-> buildApkWithV1sign]

```java
public void buildApkWithV1sign(HashMap<String, Integer> compressData) throws IOException, InterruptedException {
    insureFileNameV1();
    generalUnsignApk(compressData);
    signApkV1(mUnSignedApk, mSignedApk);
    use7zApk(compressData, mSignedApk, mSignedWith7ZipApk);
    alignApks();
    copyFinalApkV1();
  }
```

## [ResourceApkBuilder-> insureFileNameV1]

```java
private void insureFileNameV1() {
    mUnSignedApk = new File(mOutDir.getAbsolutePath(), mApkName + "_unsigned.apk");
    mSignedWith7ZipApk = new File(mOutDir.getAbsolutePath(), mApkName + "_signed_7zip.apk");
    mSignedApk = new File(mOutDir.getAbsolutePath(), mApkName + "_signed.apk");
    mAlignedApk = new File(mOutDir.getAbsolutePath(), mApkName + "_signed_aligned.apk");
    mAlignedWith7ZipApk = new File(mOutDir.getAbsolutePath(), mApkName + "_signed_7zip_aligned.apk");
    m7zipOutPutDir = new File(mOutDir.getAbsolutePath(), TypedValue.OUT_7ZIP_FILE_PATH);
  }
```

* 这个方法其实就是构建出构建所需的所有文件

## [ResourceApkBuilder-> generalUnsignApk]

```java
File tempOutDir = new File(mOutDir.getAbsolutePath(), TypedValue.UNZIP_FILE_PATH);
File[] unzipFiles = tempOutDir.listFiles();
    assert unzipFiles != null;
    List<File> collectFiles = new ArrayList<>();
    for (File f : unzipFiles) {
      String name = f.getName();
      if (name.equals("res") || name.equals("resources.arsc")) {
        continue;
      } else if (name.equals(config.mMetaName)) {
        addNonSignatureFiles(collectFiles, f);
        continue;
      }
      collectFiles.add(f);
    }
```

* 这里可以看到首先会构建一个临时文件，然后把遍历解压文件，如果是res目录或者是resources.arsc就不予处理
* 如果遇到的是签名文件夹，就调用**addNonSignatureFiles**进行操作
* 最终都会把除了res目录和resources.arsc以及的文件保存在**collectFiles**中，

## [ResourceApkBuilder-> addNonSignatureFiles]

```java
private void addNonSignatureFiles(List<File> collectFiles, File metaFolder) {
    File[] metaFiles = metaFolder.listFiles();
    if (metaFiles != null) {
      for (File metaFile : metaFiles) {
        String metaFileName = metaFile.getName();
        // Ignore signature files
        if (!metaFileName.endsWith(".MF") && !metaFileName.endsWith(".RSA") && !metaFileName.endsWith(".SF")) {
          System.out.println(String.format("add meta file %s", metaFile.getAbsolutePath()));
          collectFiles.add(metaFile);
        }
      }
    }
  }
```

* 在这个方法里面，其实**collectFiles**添加进去的也是和签名无关的文件
* 所以总结起来**collectFiles**这个容器中存放的就是除**签名**，**arsc**，**res**之外的所有文件
* 剩下就是看一下什么时候用到这个**collectFiles**了

## [ResourceApkBuilder-> generalUnsignApk]

```java
 File destResDir = new File(mOutDir.getAbsolutePath(), "res");
    //添加修改后的res文件
    if (!config.mKeepRoot && FileOperation.getlist(destResDir) == 0) {
      destResDir = new File(mOutDir.getAbsolutePath(), TypedValue.RES_FILE_PATH);
    }
 File rawResDir = new File(tempOutDir.getAbsolutePath() + File.separator + "res");
 collectFiles.add(destResDir);
 File rawARSCFile = new File(mOutDir.getAbsolutePath() + File.separator + "resources.arsc");
 collectFiles.add(rawARSCFile);
 FileOperation.zipFiles(collectFiles, tempOutDir, mUnSignedApk, compressData);
```

* 这里其实说实话一开始看是有点懵的，这主要是因为文件的关系没有理清，其实这个作者我觉得做得是挺高明的一点就是避免了参数传递，因为全过程中涉及文件的操作比较多，如果把确认好的文件到处传这样方法参数比较臃肿，我猜想这个作者当时的想法应该就是全局制定一些条约，根据文件名，去在不同的类中拿到想要的文件
* **destResDir：**这个就是混淆之后的r文件夹，这里这个判断的就是判断res文件夹里是否有文件，有的话就说明原来储存的资源的文件夹就是res，也就是是根目录不混淆的情况，如果没有的话，就说明根目录就是r，就是经过混淆的情况
* 然后就把**resources.arsc**和**destResDir**添加到之前的**collectFiles**中
* 然后就意味着apk的文件已经收集完成，就进行压缩操作，在这个压缩操作中需要4个参数，**收集完成的apk文件**，**解压目录**，**一开始设定好的UnSignedApk文件**，极度关键的**compressData**，关注它是如何实现压缩的

## [FileOperation-> zipFiles]

```java
public static void zipFiles(
      Collection<File> resFileList, File baseFolder, File zipFile, HashMap<String, Integer> compressData)
      throws IOException {
    ZipOutputStream zipOut = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(zipFile), BUFFER));
    for (File resFile : resFileList) {
      if (resFile.exists()) {
        if (resFile.getAbsolutePath().contains(baseFolder.getAbsolutePath())) {
          String relativePath = baseFolder.toURI().relativize(resFile.getParentFile().toURI()).getPath();
          // remove slash at end of relativePath
          if (relativePath.length() > 1) {
            relativePath = relativePath.substring(0, relativePath.length() - 1);
          } else {
            relativePath = "";
          }
          zipFile(resFile, zipOut, relativePath, compressData);
        } else {
          zipFile(resFile, zipOut, "", compressData);
        }
      }
    }
    zipOut.close();
  }
```

* 这个方法暂时没用到**compressData**
* 但是这里拿到了一个**relativePath**，这个**relativePath**是什么？就是当前文件和源目录底下相隔的目录字符串

## [FileOperation-> zipFile]

```java
private static void zipFile(
      File resFile, ZipOutputStream zipout, String rootpath, HashMap<String, Integer> compressData) throws IOException {
    rootpath = rootpath + (rootpath.trim().length() == 0 ? "" : File.separator) + resFile.getName();
    if (resFile.isDirectory()) {
      File[] fileList = resFile.listFiles();
      for (File file : fileList) {
        zipFile(file, zipout, rootpath, compressData);
      }
    } else {
      final byte[] fileContents = readContents(resFile);
      //这里需要强转成linux格式，果然坑！！
      if (rootpath.contains("\\")) {
        rootpath = rootpath.replace("\\", "/");
      }
      if (!compressData.containsKey(rootpath)) {
        System.err.printf(String.format("do not have the compress data path =%s in resource.asrc\n", rootpath));
        //throw new IOException(String.format("do not have the compress data path=%s", rootpath));
        return;
      }
      int compressMethod = compressData.get(rootpath);
      ZipEntry entry = new ZipEntry(rootpath);

      if (compressMethod == ZipEntry.DEFLATED) {
        entry.setMethod(ZipEntry.DEFLATED);
      } else {
        entry.setMethod(ZipEntry.STORED);
        entry.setSize(fileContents.length);
        final CRC32 checksumCalculator = new CRC32();
        checksumCalculator.update(fileContents);
        entry.setCrc(checksumCalculator.getValue());
      }
      zipout.putNextEntry(entry);
      zipout.write(fileContents);
      zipout.flush();
      zipout.closeEntry();
    }
  }
```

* 首先把资源文件的内存读出来保存在byte数组**fileContent**中
* 可以看到，这里的逻辑就是判断**compressData**是否包含**rootpath**，如果没有就直接返回了
* 读到这里有个问题，这个**compressData**是什么时候put东西进去的？目测是之前分析resources.arsc读取流程的时候漏掉了

## [ARSCDecoder-> readValue]

```java
//这里用的是linux的分隔符
        HashMap<String, Integer> compressData = mApkDecoder.getCompressData();
        if (compressData.containsKey(raw)) {
          compressData.put(result, compressData.get(raw));
        } else {
          System.err.printf("can not find the compress dataresFile=%s\n", raw);
        }
```

* 继续跟踪回**mApkDecoder**中的**compressData**

## [ApkDecoder-> ensureFilePath]

```java
private void ensureFilePath() throws IOException {
   mCompressData = FileOperation.unZipAPk(apkFile.getAbsoluteFile().getAbsolutePath(), unZipDest);
   dealWithCompressConfig();
}
```

## [FileOperation-> unZipAPk]

```java
public static HashMap<String, Integer> unZipAPk(String fileName, String filePath) throws IOException {
    checkDirectory(filePath);
    ZipFile zipFile = new ZipFile(fileName);
    Enumeration emu = zipFile.entries();
    HashMap<String, Integer> compress = new HashMap<>();
    try {
      while (emu.hasMoreElements()) {
        ZipEntry entry = (ZipEntry) emu.nextElement();
        if (entry.isDirectory()) {
          new File(filePath, entry.getName()).mkdirs();
          continue;
        }
        BufferedInputStream bis = new BufferedInputStream(zipFile.getInputStream(entry));

        File file = new File(filePath + File.separator + entry.getName());

        File parent = file.getParentFile();
        if (parent != null && (!parent.exists())) {
          parent.mkdirs();
        }
        //要用linux的斜杠
        String compatibaleresult = entry.getName();
        if (compatibaleresult.contains("\\")) {
          compatibaleresult = compatibaleresult.replace("\\", "/");
        }
        compress.put(compatibaleresult, entry.getMethod());
        FileOutputStream fos = new FileOutputStream(file);
        BufferedOutputStream bos = new BufferedOutputStream(fos, BUFFER);

        byte[] buf = new byte[BUFFER];
        int len;
        while ((len = bis.read(buf, 0, BUFFER)) != -1) {
          fos.write(buf, 0, len);
        }
        bos.flush();
        bos.close();
        bis.close();
      }
    } finally {
      zipFile.close();
    }
    return compress;
  }
```

* 首先第一步对输出目录进行检查，如果存在就删除重建
* 然后遍历apk中的Entry
* 在**compress**里面存的是**修改后的entryName**对应**entry压缩方法**的映射

## [ApkDecoder-> dealWithCompressConfig]

```java
private void dealWithCompressConfig() {
    if (config.mUseCompress) {
      HashSet<Pattern> patterns = config.mCompressPatterns;
      if (!patterns.isEmpty()) {
        for (Entry<String, Integer> entry : mCompressData.entrySet()) {
          String name = entry.getKey();
          for (Iterator<Pattern> it = patterns.iterator(); it.hasNext(); ) {
            Pattern p = it.next();
            if (p.matcher(name).matches()) {
              mCompressData.put(name, TypedValue.ZIP_DEFLATED);
            }
          }
        }
      }
    }
  }
```

* 看到这里真的有一种神清气爽的感觉
* 这里第一步就是从config中拿到**mCompressPatterns**，而config中的**mCompressPatterns**的来源就是来自gradle extension
* 然后接下来只需要遍历一次**mCompressData**，然后把获得的**entry**名称和正则表达式对比，如果是match的，那就put进**ZIP_DEFLATED**这个value
* 所以实际上压缩就是吧entry的**compressMethod**设置成**TypedValue.ZIP_DEFLATED**，也就是说其实**AndResGuard**压缩的本质就是把压缩方式设置成**DEFLATED**，让它可以压缩存储，而不是打包归档

## [ResourceApkBuilder-> generalUnsignApk]

## [ResourceApkBuilder-> buildApkWithV1sign]

## [ResourceApkBuilder-> signApkV1]

```java
private void signApkV1(File unSignedApk, File signedApk) throws IOException, InterruptedException {
    if (config.mUseSignAPK) {
      System.out.printf("signing apk: %s\n", signedApk.getName());
      if (signedApk.exists()) {
        signedApk.delete();
      }
      signWithV1sign(unSignedApk, signedApk);
      if (!signedApk.exists()) {
        throw new IOException("Can't Generate signed APK. Plz check your v1sign info is correct.");
      }
    }
  }
```

## [ResourceApkBuilder-> signWithV1sign]

```java
private void signWithV1sign(File unSignedApk, File signedApk) throws IOException, InterruptedException {
    String signatureAlgorithm = "MD5withRSA";
    try {
      signatureAlgorithm = getSignatureAlgorithm(config.digestAlg);
    } catch (Exception e) {
      e.printStackTrace();
    }
    String[] argv = {
        "jarsigner",
        "-sigalg",
        signatureAlgorithm,
        "-digestalg",
        config.digestAlg,
        "-keystore",
        config.mSignatureFile.getAbsolutePath(),
        "-storepass",
        config.mStorePass,
        "-keypass",
        config.mKeyPass,
        "-signedjar",
        signedApk.getAbsolutePath(),
        unSignedApk.getAbsolutePath(),
        config.mStoreAlias
    };
    Utils.runExec(argv);
  }
```

* 可以看到签名的方式就是采用命令行工具执行**jarsigner**命令

## [ResourceApkBuilder-> buildApkWithV1sign]

## [ResourceApkBuilder-> use7zApk]

```java
private boolean use7zApk(HashMap<String, Integer> compressData, File originalAPK, File outputAPK)
      throws IOException, InterruptedException {
    
    FileOperation.unZipAPk(originalAPK.getAbsolutePath(), m7zipOutPutDir.getAbsolutePath());
    //首先一次性生成一个全部都是压缩的安装包
    generalRaw7zip(outputAPK);

    ArrayList<String> storedFiles = new ArrayList<>();

    for (String name : compressData.keySet()) {
      File file = new File(m7zipOutPutDir.getAbsolutePath(), name);
      if (!file.exists()) {
        continue;
      }
      int method = compressData.get(name);
      if (method == TypedValue.ZIP_STORED) {
        storedFiles.add(name);
      }
    }

    addStoredFileIn7Zip(storedFiles, outputAPK);
    return true;
  }
```

* 首先第一步就是把原本的签名apk解压
* 调用**generalRaw7zip**方法
* 对于不压缩的文件文件用一个容器储存起来
* 调用**addStoredFileIn7Zip**方法

## [ResourceApkBuilder-> generalRaw7zip]

```java
private void generalRaw7zip(File outSevenZipApk) throws IOException, InterruptedException {
    String outPath = m7zipOutPutDir.getAbsoluteFile().getAbsolutePath();
    String path = outPath + File.separator + "*";
    String cmd = Utils.isPresent(config.m7zipPath) ? config.m7zipPath : TypedValue.COMMAND_7ZIP;
    Utils.runCmd(cmd, "a", "-tzip", outSevenZipApk.getAbsolutePath(), path, "-mx9");
  }
```

* 采用命令行工具实现7zip压缩

## [ResourceApkBuilder-> addStoredFileIn7Zip]

```java
private void addStoredFileIn7Zip(ArrayList<String> storedFiles, File outSevenZipAPK)
      throws IOException, InterruptedException {

    String storedParentName = mOutDir.getAbsolutePath() + File.separator + "storefiles" + File.separator;
    String outputName = m7zipOutPutDir.getAbsolutePath() + File.separator;
    for (String name : storedFiles) {
      FileOperation.copyFileUsingStream(new File(outputName + name), new File(storedParentName + name));
    }
    storedParentName = storedParentName + File.separator + "*";
    String cmd = Utils.isPresent(config.m7zipPath) ? config.m7zipPath : TypedValue.COMMAND_7ZIP;
    Utils.runCmd(cmd, "a", "-tzip", outSevenZipAPK.getAbsolutePath(), storedParentName, "-mx0");
  }
```

* 这里的任务就是把刚刚设计stored压缩方式的文件添加到7zip压缩包中

## [ResourceApkBuilder-> buildApkWithV1sign]

## [ResourceApkBuilder-> alignApks]

```java
private void alignApks() throws IOException, InterruptedException {
    if (mSignedApk.exists()) {
      alignApk(mSignedApk, mAlignedApk);
    }
    if (mSignedWith7ZipApk.exists()) {
      alignApk(mSignedWith7ZipApk, mAlignedWith7ZipApk);
    }
  }
```

## [ResourceApkBuilder-> alignApk]

```java
 private void alignApk(File before, File after) throws IOException, InterruptedException {
    String cmd = Utils.isPresent(config.mZipalignPath) ? config.mZipalignPath : TypedValue.COMMAND_ZIPALIGIN;
   Utils.runCmd(cmd, "4", before.getAbsolutePath(), after.getAbsolutePath());
  }
```

* 也是使用命令行工具进行apk对齐

## 阶段小结

整个v1签名的apk生成的过程走完了，总结起来这里包括align和7zip还有v1签名都是采用命令行工具完成的，而压缩的话则是通过设置ZipEntry的method实现的

## [Main-> buildApk]

## [ResourceApkBuilder-> buildApkWithV2sign]

```java
public void buildApkWithV2sign(HashMap<String, Integer> compressData, int minSDKVersion) throws Exception {
    insureFileNameV2();
    generalUnsignApk(compressData);
    if (use7zApk(compressData, mUnSignedApk, m7ZipApk)) {
      alignApk(m7ZipApk, mAlignedApk);
    } else {
      alignApk(mUnSignedApk, mAlignedApk);
    }

    /*
     * Caution: If you sign your app using APK Signature Scheme v2 and make further changes to the app,
     * the app's signature is invalidated.
     * For this reason, use tools such as zipalign before signing your app using APK Signature Scheme v2, not after.
     **/
    signApkV2(mAlignedApk, mSignedApk, minSDKVersion);
    copyFinalApkV2();
  }

```

* v2签名的处理流程和v1签名的处理流程大同小异
* 不一样的是v2签名的逻辑中**align**的操作是放在v2签名之前的，这是因为v2签名保护的是zip文件的1、3、4部分后如果进行align操作的话就会被视为改变的apk结构，然后就会签名校验失败

## [ResourceApkBuilder-> signApkV2]

## [ResourceApkBuilder-> signWithV2sign]

```java
private void signWithV2sign(File unSignedApk, File signedApk, int minSDKVersion) throws Exception {
    String[] params = new String[] {
        "sign",
        "--ks",
        config.mSignatureFile.getAbsolutePath(),
        "--ks-pass",
        "pass:" + config.mStorePass,
        "--min-sdk-version",
        String.valueOf(minSDKVersion),
        "--ks-key-alias",
        config.mStoreAlias,
        "--key-pass",
        "pass:" + config.mKeyPass,
        "--out",
        signedApk.getAbsolutePath(),
        unSignedApk.getAbsolutePath()
    };
    ApkSignerTool.main(params);
  }
```

* 采用android开源的ApkSignerTool实现v2签名

## 阶段小结

来到这里，其实整个构建apk的流程都走完，但是这篇文章还没有结束，因为其实还有一个部分是没有看的**AndResGuard**是如何实现资源合并的

## [ARSCDecoder-> readValue]

```java
MergeDuplicatedResInfo filterInfo = null;
        boolean mergeDuplicatedRes = mApkDecoder.getConfig().mMergeDuplicatedRes;
        if (mergeDuplicatedRes) {
          filterInfo = mergeDuplicated(resRawFile, resDestFile, compatibaleraw, result);
          if (filterInfo != null) {
            resDestFile = new File(filterInfo.filePath);
            result = filterInfo.fileName;
          }
        }
```

## [ApkDecoder-> mergeDuplicated]

```java
private MergeDuplicatedResInfo mergeDuplicated(File resRawFile, File resDestFile, String compatibaleraw, String result) throws IOException {
    MergeDuplicatedResInfo filterInfo = null;
    // 1
    List<MergeDuplicatedResInfo> mergeDuplicatedResInfoList = mMergeDuplicatedResInfoData.get(resRawFile.length());
    if (mergeDuplicatedResInfoList != null) {
       // ......
    }else {
      MergeDuplicatedResInfo info = new MergeDuplicatedResInfo.Builder()
              .setFileName(result)
              .setFilePath(resDestFile.getAbsolutePath())
              .setOriginalName(compatibaleraw)
              .create();
      info.fileName = result;
      info.filePath = resDestFile.getAbsolutePath();
      info.originalName = compatibaleraw;

      if (mergeDuplicatedResInfoList == null) {
        mergeDuplicatedResInfoList = new ArrayList<>();
        mMergeDuplicatedResInfoData.put(resRawFile.length(), mergeDuplicatedResInfoList);
      }
      mergeDuplicatedResInfoList.add(info);
    }
    return filterInfo;
```

* 注释1的地方有一个成员**mMergeDuplicatedResInfoData**，但是所有的get和put的操作都是在这个方式执行的，所以这里肯定是不命中的，所以先看不命中的逻辑
* 首先是构建一个**MergeDuplicatedResInfo**对象，但是这里有点迷，不知道为什么builder模式要设置一次值然后后面又设置一次，然后就是把这个对象存进一个list中
* 然后就根据原来的资源文件的大小作为key，存放进**mMergeDuplicatedResInfoData**这个成员里面
* 所以很明显，文件长度是一个判断重复的其中一个标准

```java
if (mergeDuplicatedResInfoList != null) {
      for (MergeDuplicatedResInfo mergeDuplicatedResInfo : mergeDuplicatedResInfoList) {
        if (mergeDuplicatedResInfo.md5 == null) {
          mergeDuplicatedResInfo.md5 = Md5Util.getMD5Str(new File(mergeDuplicatedResInfo.filePath));
        }
        String resRawFileMd5 = Md5Util.getMD5Str(resRawFile);
        if (!resRawFileMd5.isEmpty() && resRawFileMd5.equals(mergeDuplicatedResInfo.md5)) {
          filterInfo = mergeDuplicatedResInfo;
          filterInfo.md5 = resRawFileMd5;
          break;
        }
      }
    }
    if (filterInfo != null) {
      generalFilterResIDMapping(compatibaleraw, result, filterInfo.originalName, filterInfo.fileName, resRawFile.length());
      mMergeDuplicatedResCount++;
      mMergeDuplicatedResTotalSize += resRawFile.length();
```

* 这个就是缓存命中的的情况，它的逻辑其实也就是计算当前文件的md5的值以及和当前文件长度一致的文件列表的md5的值，如果是一致的就说明是重复了
* 然后剩下的就是一些mapping写入和size记录的工作了
* 所以资源查重的标准是两个点，一个是文件长度是否一致，另一个就是md5的值是否相同







