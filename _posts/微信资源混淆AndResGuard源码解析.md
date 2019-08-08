---
layout:     post   				    
title:      微信资源混淆AndResGuard源码解析(一)	 
subtitle:   -apk学习系列
            -AndResGuard之解析resource.arsc    #副标题
date:       2019-8-8		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-map.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - apk学习系列
---

# 微信资源混淆AndResGuard源码解析(一)

## 前言

之前受到了AndResGuard激发开发出一款符合业务场景的多渠道打包的gradle-plugin，但是对于核心的resources.arsc文件的操作，并没有得到满意，所以想回顾和继续学习源码细节

## [AndResGuardPlugin-> apply]

```groovy
@Override
  void apply(Project project) {
    project.apply plugin: 'com.google.osdetector'
    project.extensions.create('andResGuard', AndResGuardExtension)
    project.extensions.add("sevenzip", new ExecutorExtension("sevenzip"))

    project.afterEvaluate {
      def android = project.extensions.android
      createTask(project, USE_APK_TASK_NAME)

      android.applicationVariants.all { variant ->
        def variantName = variant.name.capitalize()
        createTask(project, variantName)
      }

      android.buildTypes.all { buildType ->
        def buildTypeName = buildType.name.capitalize()
        createTask(project, buildTypeName)
      }

      android.productFlavors.all { flavor ->
        def flavorName = flavor.name.capitalize()
        createTask(project, flavorName)
      }

      project.extensions.findByName("sevenzip").loadArtifact(project)
    }
  }
```

## [AndResGuardTask-> run]

```groovy
@TaskAction
  run() {
  
    buildConfigs.each { config ->
          if (StringUtil.isBlank(configuration.sourceFlavor) || (StringUtil.isPresent(configuration.sourceFlavor) &&
              config.flavors.size() >0 && config.flavors.get(0).name ==
              configuration.sourceFlavor)) {
            RunGradleTask(config, configuration.sourceApk, config.minSDKVersion)
          }
        }
      } else {
        RunGradleTask(config, config.file.getAbsolutePath(), config.minSDKVersion)
      }
    }
  }
```

## [AndResGuardTask-> RunGradleTask]

```java
def RunGradleTask(config, String absPath, int minSDKVersion) {
    def signConfig = config.signConfig
    String packageName = config.packageName
    ArrayList<String> whiteListFullName = new ArrayList<>()
    ExecutorExtension sevenzip = project.extensions.findByName("sevenzip") as ExecutorExtension
    configuration.whiteList.each { res ->
      if (res.startsWith("R")) {
        whiteListFullName.add(packageName + "." + res)
      } else {
        whiteListFullName.add(res)
      }
    }

    InputParam.Builder builder = new InputParam.Builder()
        .setMappingFile(configuration.mappingFile)
        .setWhiteList(whiteListFullName)
        .setUse7zip(configuration.use7zip)
        .setMetaName(configuration.metaName)
        .setKeepRoot(configuration.keepRoot)
        .setMergeDuplicatedRes(configuration.mergeDuplicatedRes)
        .setCompressFilePattern(configuration.compressFilePattern)
        .setZipAlign(getZipAlignPath())
        .setSevenZipPath(sevenzip.path)
        .setOutBuilder(useFolder(config.file))
        .setApkPath(absPath)
        .setUseSign(configuration.useSign)
        .setDigestAlg(configuration.digestalg)
        .setMinSDKVersion(minSDKVersion)

    if (configuration.finalApkBackupPath != null && configuration.finalApkBackupPath.length() > 0){
      builder.setFinalApkBackupPath(configuration.finalApkBackupPath)
    } else {
      builder.setFinalApkBackupPath(absPath)
    }

    if (configuration.useSign) {
      if (signConfig == null) {
        throw new GradleException("can't the get signConfig for release build")
      }
      builder.setSignFile(signConfig.storeFile)
          .setKeypass(signConfig.keyPassword)
          .setStorealias(signConfig.keyAlias)
          .setStorepass(signConfig.storePassword)
      if (signConfig.hasProperty('v2SigningEnabled') && signConfig.v2SigningEnabled) {
        builder.setSignatureType(InputParam.SignatureType.SchemaV2)
      }
    }
    InputParam inputParam = builder.create()
    Main.gradleRun(inputParam)
  }
```

* 往白名单的List中添加资源的全称
* builder模式构建入参，把gradle extension中获得的参数，以及上一步处理的白名单资源全参构建起来
* 最后就是调用Main的静态方法把参数传到Main中执行
* 这种设计特点就是核心的功能和代码都是用java实现的，然后对外交互就是用gradle实现，最后用gradle task调用java的入口执行关键代码

## [Main-> gradleRun]

```java
public static void gradleRun(InputParam inputParam) {
    Main m = new Main();
    m.run(inputParam);
  }
```

## [Main-> run]

```java
private void run(InputParam inputParam) {
    synchronized (Main.class) {
      loadConfigFromGradle(inputParam);
      this.mFinalApkBackPath = inputParam.finalApkBackupPath;
      File finalApkFile = StringUtil.isPresent(inputParam.finalApkBackupPath) ?
          new File(inputParam.finalApkBackupPath)
          : null;

      resourceProguard(
          new File(inputParam.outFolder),
          finalApkFile,
          inputParam.apkPath,
          inputParam.signatureType,
          inputParam.minSDKVersion
      );
      clean();
    }
  }
```

* loadConfigFromGradle方法就是通过入参inputParams去构建一个Configuration对象，创建一些数据结构供后续使用
* 校验源apk是否存在
* 执行资源混淆

## [Main-> resourceProguard]

```java
protected void resourceProguard(
      File outputDir, File outputFile, String apkFilePath, InputParam.SignatureType signatureType, int minSDKVersoin) {
    mRawApkSize = FileOperation.getFileSizes(apkFile); 
    ApkDecoder decoder = new ApkDecoder(config, apkFile);
    /* 默认使用V1签名 */
    decodeResource(outputDir, decoder, apkFile);
    buildApk(decoder, apkFile, outputFile, signatureType, minSDKVersoin);
}
```

* 解析资源
* 构建apk

## [Main-> decodeResource]

```java
private void decodeResource(File outputFile, ApkDecoder decoder, File apkFile)
      throws AndrolibException, IOException, DirectoryException {
    if (outputFile == null) {
      mOutDir = new File(mRunningLocation, apkFile.getName().substring(0, apkFile.getName().indexOf(".apk")));
    } else {
      mOutDir = outputFile;
    }
    decoder.setOutDir(mOutDir.getAbsoluteFile());
    decoder.decode();
  }
```

* 构建一个输出目录，设置到ApkDecoder中
* 执行decoder的解析工作

## [Main-> decode]

```java
public void decode() throws AndrolibException, IOException, DirectoryException {
    if (hasResources()) {
      ensureFilePath(); // 1
      // read the resources.arsc checking for STORED vs DEFLATE compression
      // this will determine whether we compress on rebuild or not.
      System.out.printf("decoding resources.arsc\n");
      RawARSCDecoder.decode(apkFile.getDirectory().getFileInput("resources.arsc"));
      ResPackage[] pkgs = ARSCDecoder.decode(apkFile.getDirectory().getFileInput("resources.arsc"), this);
                           
      copyOtherResFiles(); // 2

      ARSCDecoder.write(apkFile.getDirectory().getFileInput("resources.arsc"), this, pkgs);
    }
  }
```

* **注释1：**在输出目录下初始化各种目录，比如res -> r，比如mapping以及arsc等文件
* 然后这里从方法名可以看到有两次ARSC的decode操作
* **注释2：**把没有记录在resources.arsc的资源文件进行拷贝，也就是res/raw文件夹中的资源拷贝过去
* 写回resource.arsc文件

## [RawARSCDecoder-> decode]

```java
public static ResPackage[] decode(InputStream arscStream) throws AndrolibException {
    try {
      RawARSCDecoder decoder = new RawARSCDecoder(arscStream);
      System.out.printf("parse to get the exist names in the resouces.arsc first\n");
      return decoder.readTable();
    } catch (IOException ex) {
      throw new AndrolibException("Could not decode arsc file", ex);
    }
  }
```

* 从注释就可以知道，这次读取resource.arsc的目的是为了得到已经存在于resource.arsc的资源名称，以下只细数读取resource.arsc文件时的细节操作

## [RawARSCDecoder-> readTablePackage]

```java
private ResPackage readTablePackage() throws IOException, AndrolibException {
   
    // TypeIdOffset was added platform_frameworks_base/@f90f2f8dc36e7243b85e0b6a7fd5a590893c827e
    // which is only in split/new applications.
    int splitHeaderSize = (2 + 2 + 4 + 4 + (2 * 128) + (4 * 5)); // short, short, int, int, char[128], int * 4
    if (mHeader.headerSize == splitHeaderSize) {
      mTypeIdOffset = mIn.readInt();
    }

    mResId = id << 24;

    return mPkg;
  }
```

* 注释的地方标注的情况是split/new applications，这种情况下header会多出一个4字节的字段typeIdOffset
* 其次就是会用mResId的高4位记录下packageId，这里其实可以猜测到应该是和后面用来和资源绑定

## [RawARSCDecoder-> readSingleTableTypeSpec]

```java
private void readSingleTableTypeSpec() throws AndrolibException, IOException {
    int id = mIn.readUnsignedByte();
    mCurTypeID = id;
    mResId = (0xff000000 & mResId) | id << 16;
    mType = new ResType(mTypeNames.getString(id - 1), mPkg);
  }
```

* 当在读取TypeSpec的时候，用一个成员变量记录下上面读出的id值
* 在ResId中的次4位添加上类型信息
* mType记录下当前读到的资源类型，包含资源类型的名称和对应的包

## [RawARSCDecoder-> readConfig]

```java
private void readConfig() throws IOException, AndrolibException {
  
    int[] entryOffsets = mIn.readIntArray(entryCount);
    for (int i = 0; i < entryOffsets.length; i++) {
      if (entryOffsets[i] != -1) {
        mResId = (mResId & 0xffff0000) | i;
        readEntry();
      }
    }
  }
```

* 最终在mResId中添加上entry对应的偏移量
* 然后进入entry的读取

## [RawARSCDecoder-> readEntry]

```java
private void readEntry() throws IOException, AndrolibException {
    putTypeSpecNameStrings(mCurTypeID, mSpecNames.getString(specNamesId));
    boolean readDirect = false;
    if ((flags & ENTRY_FLAG_COMPLEX) == 0) {
      readDirect = true;
      readValue(readDirect, specNamesId);
    } else {
      readDirect = false;
      readComplexEntry(readDirect, specNamesId);
    }
  }
```

* 在entry读取的阶段，调用了**putTypeSpecNameStrings**方法把之前记录下来的TypeId和之前在读取package阶段拿到的资源项名称字符串池中的对应名称
* 在看**putTypeSpecNameStrings**这个方法之前其实到目前为止的步骤是存疑的，首先第一点是mResId的线索断了，第二点是**readDirect**方法的两个入参其实完全没有被用到

## [RawARSCDecoder-> putTypeSpecNameStrings]

```java
private void putTypeSpecNameStrings(int type, String name) {
    Set<String> names = mExistTypeNames.get(type);
    if (names == null) {
      names = new HashSet<>();
    }
    names.add(name);
    mExistTypeNames.put(type, names);
  }
```

* 这个逻辑其实就是构建一种数据结构把资源类型和此资源类型中的资源名称的集合关联起来，猜测后面混淆的时候会用到这个映射关系

## 阶段小结

到目前为止的这个阶段，可以看到在第一次去读取resource.arsc文件的过程中，核心的逻辑其实就是建立一个资源类型和资源类型中资源名称集合的map，应该是供后面混淆的时候使用，答案应该就在第二次进行resource.arsc文件的过程中

## [ARSCDecoder-> decode]

```java
public static ResPackage[] decode(InputStream arscStream, ApkDecoder apkDecoder) throws AndrolibException {
    try {
      ARSCDecoder decoder = new ARSCDecoder(arscStream, apkDecoder);
      ResPackage[] pkgs = decoder.readTable();
      return pkgs;
    } catch (IOException ex) {
      throw new AndrolibException("Could not decode arsc file", ex);
    }
  }
```

## [ARSCDecoder-> readTable]

```java
private ResPackage[] readTable() throws IOException, AndrolibException {
    nextChunkCheckType(Header.TYPE_TABLE);
    int packageCount = mIn.readInt();
    mTableStrings = StringBlock.read(mIn);
    ResPackage[] packages = new ResPackage[packageCount];
    nextChunk();
    for (int i = 0; i < packageCount; i++) {
      packages[i] = readPackage();
    }
    // 1
    mMappingWriter.close();
    System.out.printf("resources mapping file %s done\n", mApkDecoder.getResMappingFile().getAbsolutePath());
    generalFilterEnd(mMergeDuplicatedResCount, mMergeDuplicatedResTotalSize);
    mMergeDuplicatedResMappingWriter.close();
    System.out.printf("resources filter mapping file %s done\n", mApkDecoder.getMergeDuplicatedResMappingFile().getAbsolutePath());
    return packages;
  }
```

* 在**注释1**之前的逻辑其实就是读取resource.arsc中的package部分
* 而**注释1**之后的操作就是对mapping文件操作类的关闭操作，这里关注**MappingWriter**和**MergeDuplicatedResMappingWriter**，后面详细看这两个**Writer**做了什么事情

## [ARSCDecoder-> readPackage]

```java
private ResPackage readPackage() throws IOException, AndrolibException {
 
    mCurrTypeID = -1;
    mResId = id << 24;
    mPkg = new ResPackage(id, name);
    // 系统包名不混淆
    if (mPkg.getName().equals("android")) {
      mPkg.setCanResguard(false);
    } else {
      mPkg.setCanResguard(true);
    }
    nextChunk();
    while (mHeader.type == Header.TYPE_LIBRARY) {
      readLibraryType();
    }
    while (mHeader.type == Header.TYPE_SPEC_TYPE) {
      readTableTypeSpec();
    }
    return mPkg;
  }
```

* 在解析package的操作中添加了一步操作就是给这个package一个标记位，判断这个package是否可以被混淆

## [ARSCDecoder-> readTableTypeSpec]

```java
private void readTableTypeSpec() throws AndrolibException, IOException {
   
    mType = new ResType(mTypeNames.getString(id - 1), mPkg);
    // first meet a type of resource
    if (mCurrTypeID != id) {
      mCurrTypeID = id;
      initResGuardBuild(mCurrTypeID);
    }
    // 是否混淆文件路径
    mShouldResguardForType = isToResguardFile(mTypeNames.getString(id - 1));
    
    mResId = (0xff000000 & mResId) | id << 16;

    while (nextChunk().type == Header.TYPE_TYPE) {
      readConfig();
    }
  }
```

* 在typeSpec部分的读取中，添加进去的操作是用一个成员变量mCurrTypeID 记录下读到的资源类型的id，然后调用了**initResGuardBuild**方法，给资源混淆的类做一个初始化的工作
* 然后下一步就调用**isToResguardFile**通过资源类型的名称判断是否需要混淆文件路径，这个猜测是因为其实并不是所有的资源类型都会有文件路径，比如像resource.arsc中的string，id，integer等这种类型，后面进一步看他是怎么判断的

## [ ARSCDecoder-> isToResguardFile]

```java
/**
   * 为了加速，不需要处理string,id,array，这几个是肯定不是的
   */
  private boolean isToResguardFile(String name) {
    return (!name.equals("string") && !name.equals("id") && !name.equals("array"));
  }
```

- 实锤和猜想一致

## [ARSCDecoder-> initResGuardBuild]

```java
private void initResGuardBuild(int resTypeId) {
    // we need remove string from resguard candidate list if it exists in white list
    HashSet<Pattern> whiteListPatterns = getWhiteList(mType.getName());
    // init resguard builder
    mResguardBuilder.reset(whiteListPatterns);
    mResguardBuilder.removeStrings(RawARSCDecoder.getExistTypeSpecNameStrings(resTypeId));
    // 如果是保持mapping的话，需要去掉某部分已经用过的mapping
    reduceFromOldMappingFile();
  }
```

## [ARSCDecoder-> getWhiteList]

```java
private HashSet<Pattern> getWhiteList(String resType) {
    final String packName = mPkg.getName();
    if (mApkDecoder.getConfig().mWhiteList.containsKey(packName)) {
      if (mApkDecoder.getConfig().mUseWhiteList) {
        HashMap<String, HashSet<Pattern>> typeMaps = mApkDecoder.getConfig().mWhiteList.get(packName);
        return typeMaps.get(resType);
      }
    }
    return null;
  }
```

* 这里拿到了mApkDecoder中的config中的mWhiteList，但是其实在这里会比较蒙蔽，主要是因为对于mWhiteList的数据结构不太清晰
* 所以往回找，这个mWhiteList到底是在什么时候初始化的，怎么初始化的？

## [Main-> run]

## [Main-> loadConfigFromGradle]

## [Configuration-> Configuration(InputParams)]

```java
public final HashMap<String, HashMap<String, HashSet<Pattern>>> mWhiteList;
public Configuration(InputParam param) throws IOException {
    mWhiteList = new HashMap<>();
    for (String item : param.whiteList) {
      mUseWhiteList = true;
      addWhiteList(item);
    }
  }
```

* 这个**param.whiteList**就是之前在**AndResGuardTask**处理的资源全称白名单，在**AndResGuardTask**中添加了包名的信息，其实就是为下面的**addWhiteList**的操作埋下伏笔
* mWhiteList 的数据结构为 **HashMap<String, HashMap<String, HashSet<Pattern>>>**

## [Configuration-> addWhiteList]

```java
private void addWhiteList(String item) throws IOException {
    if (item.length() == 0) {
      throw new IOException("Invalid config file: Missing required attribute " + ATTR_VALUE);
    }

    int packagePos = item.indexOf(".R.");
    if (packagePos == -1) {

      throw new IOException(String.format("please write the full package name,eg com.tencent.mm.R.drawable.dfdf, but yours %s\n",
          item
      ));
    }
    //先去掉空格
    item = item.trim();
    String packageName = item.substring(0, packagePos);
    //不能通过lastDot
    int nextDot = item.indexOf(".", packagePos + 3);
    String typeName = item.substring(packagePos + 3, nextDot);
    String name = item.substring(nextDot + 1);
    HashMap<String, HashSet<Pattern>> typeMap;

    if (mWhiteList.containsKey(packageName)) {
      typeMap = mWhiteList.get(packageName);
    } else {
      typeMap = new HashMap<>();
    }

    HashSet<Pattern> patterns;
    if (typeMap.containsKey(typeName)) {
      patterns = typeMap.get(typeName);
    } else {
      patterns = new HashSet<>();
    }

    name = Utils.convertToPatternString(name);
    Pattern pattern = Pattern.compile(name);
    patterns.add(pattern);
    typeMap.put(typeName, patterns);
    System.out.println(String.format("convertToPatternString typeName %s format %s", typeName, name));
    mWhiteList.put(packageName, typeMap);
  }
```

* 首先就是对传入的白名单item进行校验，长度为0抛出异常，如果白名单item中不包含包名的信息也抛出异常
* 然后分别根据资源的全称拿到了**包名**，**资源类型名称**，**资源名称**
* mWhiteList 的数据结构为 **HashMap<String, HashMap<String, HashSet<Pattern>>>**，这个mWhiteList的第一层的key代表的是包名，而根据包名取出来的**HashMap<String, HashSet<Patten>>**的key代表的是资源的类型名称，通过资源的类型名称又拿到了一个正则表达式编译表示形式集合
* 然后就根据上面得到的**资源名称**通过一个工具类Utils去把**资源名称**转换成一个正则表达式，所以其实这里猜测**AndResGuard**是通过正则表达式的方式去匹配白名单的资，而不是直接由extension中的资源全称进行白名单匹配
* 然后就是放回容器的操作了

## [ARSCDecoder-> getWhiteList]

```java
private HashSet<Pattern> getWhiteList(String resType) {
    final String packName = mPkg.getName();
    if (mApkDecoder.getConfig().mWhiteList.containsKey(packName)) {
      if (mApkDecoder.getConfig().mUseWhiteList) {
        HashMap<String, HashSet<Pattern>> typeMaps = mApkDecoder.getConfig().mWhiteList.get(packName);
        return typeMaps.get(resType);
      }
    }
    return null;
  }
```

* 有了上面构建mWhiteList的前提下，就很确定，这里返回的HashSet就是在whiteList对应资源类型的正则表达式集合

## [ARSCDecoder-> initResGuardBuild]

```java
private void initResGuardBuild(int resTypeId) {
    // we need remove string from resguard candidate list if it exists in white list
    HashSet<Pattern> whiteListPatterns = getWhiteList(mType.getName());
    // init resguard builder
    mResguardBuilder.reset(whiteListPatterns);
    mResguardBuilder.removeStrings(RawARSCDecoder.getExistTypeSpecNameStrings(resTypeId));
    // 如果是保持mapping的话，需要去掉某部分已经用过的mapping
    reduceFromOldMappingFile();
  }
```

* 回到最开始的**init**方法，在获取了当前的资源类型的白名单正则表达式集合之后
* 下一步就是去对**mResguardBuilder**这个成员进行操作，吸收了上面的经验，这里直接调了**mResguardBuilder**这个成员变量的两个方法**reset**，**removeStrings**方法，但是先回头找一下这个**mResguardBuilder**成员初始化的地方

## [ARSCDecoder-> (InputStream arscStream, ApkDecoder decoder)]

```java
private ARSCDecoder(InputStream arscStream, ApkDecoder decoder) throws AndrolibException, IOException {
    mOldFileName = new LinkedHashMap<>();
    mCurSpecNameToPos = new LinkedHashMap<>();
    mShouldResguardTypeSet = new HashSet<>();
    mIn = new ExtDataInput(new LEDataInputStream(arscStream));
    mApkDecoder = decoder;
    proguardFileName();
  }
```

* 在构造函数这里初始化了各种信息，其中调用了**proguardFileName**方法，而**mResguardBuilder**这个成员变量就是在这里初始化的
* 除此之外在构造函数里面还初始化了1个成员变量，在**proguardFileName**中会用到的
* **mOldFileName**

## [ARSCDecoder-> proguardFileName]

```java
private void proguardFileName() throws IOException, AndrolibException {
    //-------------------------
    mMappingWriter = new BufferedWriter(new FileWriter(mApkDecoder.getResMappingFile(), false));
    mMergeDuplicatedResMappingWriter = new BufferedWriter(new FileWriter(mApkDecoder.getMergeDuplicatedResMappingFile(), false));
    mMergeDuplicatedResMappingWriter.write("res filter path mapping:\n");
    mMergeDuplicatedResMappingWriter.flush();
    //-------------------------
    mResguardBuilder = new ResguardStringBuilder();
    mResguardBuilder.reset(null);
    //-------------------------
    final Configuration config = mApkDecoder.getConfig();

    File rawResFile = mApkDecoder.getRawResFile();

    File[] resFiles = rawResFile.listFiles();

    // 需要看看哪些类型是要混淆文件路径的
    for (File resFile : resFiles) {
      String raw = resFile.getName();
      if (raw.contains("-")) {
        raw = raw.substring(0, raw.indexOf("-"));
      }
      mShouldResguardTypeSet.add(raw);
    }

    if (!config.mKeepRoot) {
      // 需要保持之前的命名方式
      if (config.mUseKeepMapping) {
        HashMap<String, String> fileMapping = config.mOldFileMapping;
        List<String> keepFileNames = new ArrayList<>();
        // 这里面为了兼容以前，也需要用以前的文件名前缀，即res混淆成什么
        String resRoot = TypedValue.RES_FILE_PATH;
        for (String name : fileMapping.values()) {
          int dot = name.indexOf("/");
          if (dot == -1) {
            throw new IOException(String.format("the old mapping res file path should be like r/a, yours %s\n", name));
          }
          resRoot = name.substring(0, dot);
          keepFileNames.add(name.substring(dot + 1));
        }
        // 去掉所有之前保留的命名，为了简单操作，mapping里面有的都去掉
        mResguardBuilder.removeStrings(keepFileNames);

        for (File resFile : resFiles) {
          String raw = "res" + "/" + resFile.getName();
          if (fileMapping.containsKey(raw)) {
            mOldFileName.put(raw, fileMapping.get(raw));
          } else {
            mOldFileName.put(raw, resRoot + "/" + mResguardBuilder.getReplaceString());
          }
        }
      } else {
        for (int i = 0; i < resFiles.length; i++) {
          // 这里也要用linux的分隔符,如果普通的话，就是r
          mOldFileName.put("res" + "/" + resFiles[i].getName(),
             TypedValue.RES_FILE_PATH + "/" + mResguardBuilder.getReplaceString()
          );
        }
      }
      generalFileResMapping();
    }

    Utils.cleanDir(mApkDecoder.getOutResFile());
  }
```

* 首先第一部分就是**mapping**文件写入类和**DuplicatedResMapping**写入类的初始化
* 第二步就是对**mResguardBuilder**这个成员变量的初始化，也就是刚刚关注的点
* 第三步就是拿到解压apk目录下res目录
* 第四步就是遍历res目录下的文件目录，看看哪些类型是要混淆文件路径的，然后用**mShouldResguardTypeSet**存起来，这里一个细节就是，在保存的时候会忽略资源差异的目录
* 然后就是一个标准标志位，mKeepRoot，进来之后又是另个标记位**mUseKeepMapping**，当不指定mappingFile的情况下这个标志位是false，这里就是为了支持增量情况
* 先看一下不适用mappingFile的情况下，就是把原本的**res/目录名**的字符串改成**r/混淆后名称**，然后记录在**mOldFileName**这个Map中，这里先留意的是**mResguardBuilder.getReplaceString()**拿到的就是更换的目录名称
* 我们看构建的最初目的就是为了观察**mResguardBuilder**这个成员变量，所以对于上面的问题先保留，先看传入mappingFile的情况
* 当在gradle中传入的mappingFile不为null的时候，这里就会拿到一个**mOldFileMapping**，这个就是把mappingFile转换而来的一个map，转换规则暂且保留，看完写入就清晰，是怎么把mappingFile转换成这个map的
* 然后下一步操作就是在**mResguardBuilder**中把mapping里有的都去掉
* 然后就还是构建一个**mOldFileName**，逻辑就是如果mapping中有，就直接用mapping中的，没有就用**mResguardBuilder.getReplaceString()**

## 阶段小结

上面构建的逻辑很复杂，在这里先记录下保留的问题，和方向

* **保留的问题：**对mappingFile转换成map的逻辑不清晰，暂时不去看的原因是因为这个极大可能和mappingFile的文件结构是有关，后续连同写入一起看
* **思路与方向：**从上几步骤开始，我们的最大的目标就是**mResguardBuilder**这个成员，回忆一下从构建到typeSpec读取的过程中这个变量做了什么事情：

```java
// ARSCDecoder构造函数---> proguardFileName
mResguardBuilder = new ResguardStringBuilder();
mResguardBuilder.reset(null);
mResguardBuilder.getReplaceString()  // 重点关注
// mapping 情况
mResguardBuilder.removeStrings(keepFileNames);
// readTableTypeSpec---> initResGuardBuild    
mResguardBuilder.reset(whiteListPatterns);
mResguardBuilder.removeStrings(RawARSCDecoder.getExistTypeSpecNameStrings(resTypeId));
```

理清思路之后就可以继续看**ResguardStringBuilder**这个类了

## [ResguardBuilder -> ()]

```java
public ResguardStringBuilder() {
      mFileNameBlackList = new HashSet<>();
      mFileNameBlackList.add("con");
      mFileNameBlackList.add("prn");
      mFileNameBlackList.add("aux");
      mFileNameBlackList.add("nul");
      mReplaceStringBuffer = new ArrayList<>();
      mIsReplaced = new HashSet<>();
      mIsWhiteList = new HashSet<>();
    }
```

## [ResguardBuilder -> reset]

```java
private String[] mAToZ = {
       "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v",
       "w", "x", "y", "z"
    };
    private String[] mAToAll = {
       "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "_", "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k",
       "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z"
    };
public void reset(HashSet<Pattern> blacklistPatterns) {
      mReplaceStringBuffer.clear();
      mIsReplaced.clear();
      mIsWhiteList.clear();

      for (int i = 0; i < mAToZ.length; i++) {
        String str = mAToZ[i];
        if (!Utils.match(str, blacklistPatterns)) {
          mReplaceStringBuffer.add(str);
        }
      }

      for (int i = 0; i < mAToZ.length; i++) {
        String first = mAToZ[i];
        for (int j = 0; j < mAToAll.length; j++) {
          String str = first + mAToAll[j];
          if (!Utils.match(str, blacklistPatterns)) {
            mReplaceStringBuffer.add(str);
          }
        }
      }

      for (int i = 0; i < mAToZ.length; i++) {
        String first = mAToZ[i];
        for (int j = 0; j < mAToAll.length; j++) {
          String second = mAToAll[j];
          for (int k = 0; k < mAToAll.length; k++) {
            String third = mAToAll[k];
            String str = first + second + third;
            if (!mFileNameBlackList.contains(str) && !Utils.match(str, blacklistPatterns)) {
              mReplaceStringBuffer.add(str);
            }
          }
        }
      }
    }
```

* reset操作首先把构造函数创建的三个容器清空掉
* 然后在构建也就是传入的参数**null**，就不停往**mReplaceStringBuffer**塞入字符串，最多支持3个字符，所以其实在初始化的时候，这个buffer中的元素数量非常多，有**26+26x37+26x37x37**个
* 而后面的情况把**whiteListPatterns**，在生成这些混淆字符串的时候，其实这样做的目的很明确了，就是为了避免白名单里的资源名称和混淆结果的字符串产生冲突，所以在这里就多了一部校验的操作，如果**混淆预备字符串**和**whiteListPatterns**冲突了，那这个**预备字符**就舍弃掉

## [ResguardBuilder -> getReplaceString]

```java
public String getReplaceString() throws AndrolibException {
      if (mReplaceStringBuffer.isEmpty()) {
        throw new AndrolibException(String.format("now can only proguard less than 35594 in a single type\n"));
      }
      return mReplaceStringBuffer.remove(0);
    }
```

## [ResguardBuilder -> removeStrings]

```java
// 对于某种类型用过的mapping，全部不能再用了
public void removeStrings(Collection<String> collection) {
      if (collection == null) return;
      mReplaceStringBuffer.removeAll(collection);
    }
```

* 这里关注到了一个小细节就是，其实在设计的时候，考虑冲突情况除了**白名单**和**混淆预备字符串**冲突这种情况之外，还有另外一种的情况就是**不在白名单的资源**和**混淆预备字符串**冲突
* 千辛万苦终于走回了**initResGuardBuild**方法

## [ARSCDecoder-> initResGuardBuild]

```java
private void initResGuardBuild(int resTypeId) {
    // we need remove string from resguard candidate list if it exists in white list
    HashSet<Pattern> whiteListPatterns = getWhiteList(mType.getName());
    // init resguard builder
    mResguardBuilder.reset(whiteListPatterns);
    mResguardBuilder.removeStrings(RawARSCDecoder.getExistTypeSpecNameStrings(resTypeId));
    // 如果是保持mapping的话，需要去掉某部分已经用过的mapping
    reduceFromOldMappingFile();
  }
```

## [ARSCDecoder-> reduceFromOldMappingFile]

```java
private void reduceFromOldMappingFile() {
    if (mPkg.isCanResguard()) {
      if (mApkDecoder.getConfig().mUseKeepMapping) {
        // 判断是否走keepmapping
        HashMap<String, HashMap<String, HashMap<String, String>>> resMapping = mApkDecoder.getConfig().mOldResMapping;
        String packName = mPkg.getName();
        if (resMapping.containsKey(packName)) {
          HashMap<String, HashMap<String, String>> typeMaps = resMapping.get(packName);
          String typeName = mType.getName();

          if (typeMaps.containsKey(typeName)) {
            HashMap<String, String> proguard = typeMaps.get(typeName);
            // 去掉所有之前保留的命名，为了简单操作，mapping里面有的都去掉
            mRes.removeStrings(proguard.values());
          }
        }
      }
    }
  }
```

* 这里的操作也是为了兼容旧的mapping
* 这里就核心操作就是在**预备混淆字符串**中移除掉mapping中有的

## 阶段小结

从上述的流程可以看到**mResguardBuilder**是混淆字符串的提供者，整个**initResGuardBuild**，就是在构建这个**预备混淆字符串**，步骤分为

* 获取白名单
* 重新设置所有**预备混淆字符串**，排除和白名单重名的字符串
* 移除和当前资源类型存在的字符串重名的
* 兼容旧的mapping，移除旧mapping中有的

**仍未解决的问题：**mapping文件的读写规则

## [ARSCDecoder-> readTableTypeSpec]

## [ARSCDecoder-> readConfig]

```java
private void readConfig() throws IOException, AndrolibException {
 
    for (int i = 0; i < entryOffsets.length; i++) {
      mCurEntryID = i;
      if (entryOffsets[i] != -1) {
        mResId = (mResId & 0xffff0000) | i;
        readEntry();
      }
    }
  }
```

* 记录下当前entry的Id
* 依然记录下资源id

## [ARSCDecoder-> readEntry]

```java
private void readEntry() throws IOException, AndrolibException {

    if (mPkg.isCanResguard()) {
      // 混淆过或者已经添加到白名单的都不需要再处理了
      if (!mResguardBuilder.isReplaced(mCurEntryID) && !mResguardBuilder.isInWhiteList(mCurEntryID)) {
        Configuration config = mApkDecoder.getConfig();
        boolean isWhiteList = false;
        if (config.mUseWhiteList) {
          isWhiteList = dealWithWhiteList(specNamesId, config);
        }

        if (!isWhiteList) {
          dealWithNonWhiteList(specNamesId, config);
        }
      }
    }

    if ((flags & ENTRY_FLAG_COMPLEX) == 0) {
      readValue(true, specNamesId);
    } else {
      readComplexEntry(false, specNamesId);
    }
  }
```

* 这里第一步就是判断当前这个entry是否已被替换过或者是否在白名单中，之前**mResguardBuilder**，存在两个成员变量**Set**分别是**mIsReplaced**和**mIsWhiteList**，因为一路代码走下来都是没有看到对这个两个容器操作的接口，所以其实这里肯定就是缓存命中的
* 然后就是判断是否使用白名单，然后就是处理白名单了

## [ARSCDecoder-> dealWithWhiteList]

```java
private boolean dealWithWhiteList(int specNamesId, Configuration config) throws AndrolibException {
    String packName = mPkg.getName();
    if (config.mWhiteList.containsKey(packName)) {
      HashMap<String, HashSet<Pattern>> typeMaps = config.mWhiteList.get(packName);
      String typeName = mType.getName();
      if (typeMaps.containsKey(typeName)) {
        String specName = mSpecNames.get(specNamesId).toString();
        HashSet<Pattern> patterns = typeMaps.get(typeName);
        for (Iterator<Pattern> it = patterns.iterator(); it.hasNext(); ) {
          Pattern p = it.next();
          if (p.matcher(specName).matches()) {
            mPkg.putSpecNamesReplace(mResId, specName);
            mPkg.putSpecNamesblock(specName);
            mResguardBuilder.setInWhiteList(mCurEntryID);

            mType.putSpecResguardName(specName);
            return true;
          }
        }
      }
    }
    return false;
  }
```

* 这里就和前面的猜测是一样的，它是采用正则表达式去匹配资源名称的
* 然后如果匹配成功就分别把资源的名称还有资源的id存储在package里面，猜测应该和写入环节是相关的
* 然后就把当前的entryId保存在mResguardBuilder中mIsWhiteList这个Set中
* 然后就把在typeSpec中也保存一个资源名称
* 然后通过返回值表明这个entry所描述的资源是否在白名单的命中

## [ARSCDecoder-> dealWithNonWhiteList]

```java
private void dealWithNonWhiteList(int specNamesId, Configuration config) throws AndrolibException, IOException {
    String replaceString = null;
    boolean keepMapping = false;
    // 1 --------------
    if (config.mUseKeepMapping) {
      String packName = mPkg.getName();
      if (config.mOldResMapping.containsKey(packName)) {
        HashMap<String, HashMap<String, String>> typeMaps = config.mOldResMapping.get(packName);
        String typeName = mType.getName();
        if (typeMaps.containsKey(typeName)) {
          HashMap<String, String> nameMap = typeMaps.get(typeName);
          String specName = mSpecNames.get(specNamesId).toString();
          if (nameMap.containsKey(specName)) {
            keepMapping = true;
            replaceString = nameMap.get(specName);
          }
        }
      }
    }

    if (!keepMapping) {
      replaceString = mResguardBuilder.getReplaceString();
    }

    // 2 -----------------
    mResguardBuilder.setInReplaceList(mCurEntryID);
    generalResIDMapping(mPkg.getName(), mType.getName(), mSpecNames.get(specNamesId).toString(), replaceString);
    mPkg.putSpecNamesReplace(mResId, replaceString);
    mPkg.putSpecNamesblock(replaceString);
    mType.putSpecResguardName(replaceString);
  }
```

* 第一部分的逻辑其实也是为了去兼容旧的mapping，从旧的mapping中拿到了上一次被混淆的资源名称
* 如果从mapping中取不到就从mResguardBuilder中获取
* 把当前的entryId保存在mResguardBuilder中的mIsReplaced这个Set中
* 往mapping中写入更换资源的映射关系
* 如处理白名单同理把资源id和对应的更换后的字符串填充到package和typeSpec中保存起来
* 依然猜测是和写入的部分有关

## [ARSCDecoder-> generalResIDMapping]

```java
private void generalResIDMapping(
     String packageName, String typename, String specName, String replace) throws IOException {
    mMappingWriter.write("    "
       + packageName
       + ".R."
       + typename
       + "."
       + specName
       + " -> "
       + packageName
       + ".R."
       + typename
       + "."
       + replace);
    mMappingWriter.write("\n");
    mMappingWriter.flush();
  }
```

* 终于看到了mMappingWriter的写入了，发现它就是一行一行的写入

## [解决疑惑：mapping的写入的规则和流程]

## [ARSCDecoder -> proguardFileName]

## [ARSCDecoder-> generalFileResMapping]

```java
private void generalFileResMapping() throws IOException {
    mMappingWriter.write("res path mapping:\n");
    for (String raw : mOldFileName.keySet()) {
      mMappingWriter.write("    " + raw + " -> " + mOldFileName.get(raw));
      mMappingWriter.write("\n");
    }
    mMappingWriter.write("\n\n");
    mMappingWriter.write("res id mapping:\n");
    mMappingWriter.flush();
  }
```

* 在这个general的过程中会把之前在构造函数中保存在mOldFileName中的**文件夹**映射关系写入进去
* 而上面的**generalResIDMapping**则是把具体的资源的映射关系写入mapping中，还是有点区别

## [ARSCDecoder-> readValue]

```java
private void readValue(boolean flags, int specNamesId) throws IOException, AndrolibException {

    //这里面有几个限制，一对于string ,id, array我们是知道肯定不用改的，第二看要那个type是否对应有文件路径
    // 1
    if (mPkg.isCanResguard()
       && flags
       && type == TypedValue.TYPE_STRING
       && mShouldResguardForType
       && mShouldResguardTypeSet.contains(mType.getName())) {
      
      // 2   
      if (mTableStringsResguard.get(data) == null) {
        String raw = mTableStrings.get(data).toString();
        String proguard = mPkg.getSpecRepplace(mResId);
        //这个要写死这个，因为resources.arsc里面就是用这个
        int secondSlash = raw.lastIndexOf("/");
        String newFilePath = raw.substring(0, secondSlash);
        if (!mApkDecoder.getConfig().mKeepRoot) {
          newFilePath = mOldFileName.get(raw.substring(0, secondSlash));
        }
       
        //同理这里不能用File.separator，因为resources.arsc里面就是用这个
        String result = newFilePath + "/" + proguard;
        int firstDot = raw.indexOf(".");
        if (firstDot != -1) {
          result += raw.substring(firstDot);
        }
        String compatibaleraw = new String(raw);
        String compatibaleresult = new String(result);

        File resRawFile = new File(mApkDecoder.getOutTempDir().getAbsolutePath() + File.separator + compatibaleraw);
        File resDestFile = new File(mApkDecoder.getOutDir().getAbsolutePath() + File.separator + compatibaleresult);
       
         //already copied
         mApkDecoder.removeCopiedResFile(resRawFile.toPath());
         mTableStringsResguard.put(data, result);
   }
 }
```

* 这个方法比较长，根据切分点一点一点慢慢看
* 首先是注释1处的判断条件，**mShouldResguardForType**就是之前排除string ,id, array的标记位，然后**mShouldResguardTypeSet**是在初始化的时候保存res下目录的集合，所以其实能猜到下面应该就是做路径混淆的工作
* 注释2：**mTableStringsResguard**一直没见到过添加的接口调用，所以这里肯定是缓存不命中的
* 然后就拿到全局常量池中全路径，然后拿到上一步的混淆字符串，从之前的缓存中拿到混淆的文件路径
* 然后就构建出新的文件路径，并且构建出两个文件resRawFile和resDestFile文件分别对应于混淆路径前和混淆路径后的文件
* 然后就在**mApkDecoder**移除已经拷贝过的资源文件
* 然后把对应value中data这个索引值和混淆后文件路径添加到**mTableStringsResguard**这个map中保存起来

## 暂时忽略

* **处理重复资源**
* **文件压缩**
* **见下一篇**

## 总结

整个读取的操作分为两次：

* **第一次读取：**把resources.arsc中已有的资源名称和关系保存起来

* **第二次读取：**把混淆路径和资源名称和原来的路径和资源名称对应起来，内存保存一份，并写入mapping

  

