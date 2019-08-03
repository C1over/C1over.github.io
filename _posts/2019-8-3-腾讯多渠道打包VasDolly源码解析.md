---
layout:     post   				    
title:      腾讯多渠道打包VasDolly源码解析		 
subtitle:   apk学习系列    #副标题
date:       2019-8-3		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-re-vs-ng2.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - apk学习系列
---

# 腾讯多渠道打包VasDolly源码解析

## 前言

上一篇文章是分析美团多渠道打包方案Walle的源码，而承前接后，想继续读一下VasDolly的源码，学习其中的设计思想还有原理

## [ApkChannelPackagePlugin-> apply]

```groovy
void apply(Project project) {

        mChannelConfigurationExtension = project.extensions.create('channel', ChannelConfigurationExtension, project) 
        mRebuildChannelConfigurationExtension = project.extensions.create('rebuildChannel', RebuildChannelConfigurationExtension, project)

        if (mProject.hasProperty(PROPERTY_CHANNELS)){
            mChanneInfolList = []
            def tempChannelsProperty = mProject.getProperties().get(PROPERTY_CHANNELS)
            if (tempChannelsProperty != null && tempChannelsProperty.trim().length() > 0) {
                tempChannelsProperty.split(",").each {
                    mChanneInfolList.add(it.trim())
                }
            }
            if (mChanneInfolList.isEmpty()){
                throw new InvalidUserDataException("Property(${PROPERTY_CHANNELS}) channel list is empty , please fix it")
            }
        }else {
            //get the channel list
            mChanneInfolList = getChannelListInfo()
        }

        project.afterEvaluate {
            project.android.applicationVariants.all { variant ->
                def variantOutput = variant.outputs.first();
                def dirName = variant.dirName;
                def variantName = variant.name.capitalize();
                Task channelTask = project.task("channel${variantName}", type: ApkChannelPackageTask) {
                    mVariant = variant;
                    mChannelExtension = mChannelConfigurationExtension;
                    mOutputDir = new File(mChannelConfigurationExtension.baseOutputDir, dirName)
                    isMergeExtensionChannelList = !mProject.hasProperty(PROPERTY_CHANNELS)
                    channelList = mChanneInfolList
                    dependsOn variant.assemble
                }
            }
        }

        project.task("reBuildChannel", type: RebuildApkChannelPackageTask) {
            isMergeExtensionChannelList = !mProject.hasProperty(PROPERTY_CHANNELS)
            channelList = mChanneInfolList
            mRebuildChannelExtension = mRebuildChannelConfigurationExtension
        }
    }

```

* 分别创建了**mChannelConfigurationExtension**和**mRebuildChannelConfigurationExtension**针对直接编译直接编译生成多渠道包和根据已有基础包重新生成多渠道包的**Extension**
* 根据**channels**这个property去获取渠道信息，没有的话便调用**getChannelListInfo**方法获取
* 然后便构建了**reBuildChannel**和**channel${variantName}**两个task

## [RebuildApkChannelPackageTask-> channel]

```groovy
@TaskAction
    public void channel() {
       
        if (mRebuildChannelExtension.isNeedRebuildDebugApk()) {
            generateChannelApk(mRebuildChannelExtension.baseDebugApk, mRebuildChannelExtension.debugOutputDir)
        }

        if (mRebuildChannelExtension.isNeedRebuildReleaseApk()) {
            generateChannelApk(mRebuildChannelExtension.baseReleaseApk, mRebuildChannelExtension.releaseOutputDir)
        }
    }
```

## [RebuildApkChannelPackageTask-> generateChannelApk]

```java
void generateChannelApk(File baseApk, File outputDir) {
        int mode = judgeChannelPackageMode(baseApk)
        if (mode == ChannelPackageTask.V1_MODE) {
           generateV1ChannelApk(baseApk, outputDir)
        } else if (mode == ChannelPackageTask.V2_MODE) {
            generateV2ChannelApk(baseApk, outputDir)
        } else {
            throw new GradleException("not have precise channel package mode")
        }
    }
```

## [RebuildApkChannelPackageTask-> judgeChannelPackageMode]

```java
int judgeChannelPackageMode(File baseApk) {
        if (ChannelReader.containV2Signature(baseApk)) {
            return ChannelPackageTask.V2_MODE
        } else if (ChannelReader.containV1Signature(baseApk)) {
            return ChannelPackageTask.V1_MODE
        } else {
            return ChannelPackageTask.DEFAULT_MODE
        }
    }
```

## [ChannelReader-> containV2Signature]

```java
public static boolean containV2Signature(File file) {
        if (file == null || !file.exists() || !file.isFile()) {
            return false;
        }
        return V2SchemeUtil.containV2Signature(file);
    }
```

## [V2SchemeUtil-> containV2Signature]

```java
public static boolean containV2Signature(File apk) {
            ByteBuffer apkSigningBlock = getApkSigningBlock(apk);
            Map<Integer, ByteBuffer> idValueMap = getAllIdValue(apkSigningBlock);
            if (idValueMap.containsKey(ApkSignatureSchemeV2Verifier.APK_SIGNATURE_SCHEME_V2_BLOCK_ID)) {
                return true;
            }
     
        return false;
    }
```

## [V2SchemeUtil-> getApkSigningBlock]

```java
 public static ByteBuffer getApkSigningBlock(File channelFile) throws ApkSignatureSchemeV2Verifier.SignatureNotFoundException, IOException {
        RandomAccessFile apk = null;
            apk = new RandomAccessFile(channelFile, "r");
            //1.find the EOCD
            Pair<ByteBuffer, Long> eocdAndOffsetInFile = ApkSignatureSchemeV2Verifier.getEocd(apk);
            ByteBuffer eocd = eocdAndOffsetInFile.getFirst();
            long eocdOffset = eocdAndOffsetInFile.getSecond();

            //2.find the APK Signing Block. The block immediately precedes the Central Directory.
            long centralDirOffset = ApkSignatureSchemeV2Verifier.getCentralDirOffset(eocd, eocdOffset);//通过eocd找到中央目录的偏移量
            //3. find the apk V2 signature block
            Pair<ByteBuffer, Long> apkSignatureBlock = ApkSignatureSchemeV2Verifier.findApkSigningBlock(apk, centralDirOffset);//找到V2签名块的内容和偏移量
            return apkSignatureBlock.getFirst();
    }
```

* 其实在这个校验签名的步骤里，感觉**Walle**和**VasDolly**大同小异，都是通过封装google的ZipUtils实现的，然后从后往前读，从EOCD开始再找到Central Diretory 再找到Signing Block，然后就拿去Signing Block中id-value数据然后返回一个HeapByteBuffer，然后退回到上一个方法**containV2Signature**去判断Signing Block里面有没有对应V2签名的id

## [RebuildApkChannelPackageTask-> judgeChannelPackageMode]

```java
int judgeChannelPackageMode(File baseApk) {
        if (ChannelReader.containV2Signature(baseApk)) {
            return ChannelPackageTask.V2_MODE    
        } else if (ChannelReader.containV1Signature(baseApk)) { // 运行位置
            return ChannelPackageTask.V1_MODE
        } else {
            return ChannelPackageTask.DEFAULT_MODE
        }
    }
```

## [ChannelReader-> containV1Signature]

```java
public static boolean containV1Signature(File file) {
        if (file == null || !file.exists() || !file.isFile()) {
            return false;
        }
        return V1SchemeUtil.containV1Signature(file);
    }
```

## [V1SchemeUtil-> containV1Signature]

```java
public static boolean containV1Signature(File file) {
        JarFile jarFile;
        try {
            jarFile = new JarFile(file);
            JarEntry manifestEntry = jarFile.getJarEntry("META-INF/MANIFEST.MF");
            JarEntry sfEntry = null;
            Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                if (entry.getName().matches("META-INF/\\w+\\.SF")) {
                    sfEntry = jarFile.getJarEntry(entry.getName());
                    break;
                }
            }
            if (manifestEntry != null && sfEntry != null) {
                return true;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return false;
    }
```

*  这个检查是否有V1签名的步骤相对没有检查V2签名的步骤麻烦，只要拿到apk中的entries，然后拿到entries中的MANIFEST.MF和.SF文件，如果两个都有，那就说明它有V1签名，当时前提就是签名已经判断过没有使用V2签名了

## [RebuildApkChannelPackageTask-> generateChannelApk]

```java
void generateChannelApk(File baseApk, File outputDir) {
        int mode = judgeChannelPackageMode(baseApk)
        if (mode == ChannelPackageTask.V1_MODE) {
           generateV1ChannelApk(baseApk, outputDir)  // 目的地
        } else if (mode == ChannelPackageTask.V2_MODE) {
            generateV2ChannelApk(baseApk, outputDir)
        } else {
            throw new GradleException("not have precise channel package mode")
        }
    }
```

## [RebuildApkChannelPackageTask-> generateV1ChannelApk]

```java
void generateV1ChannelApk(File baseApk, File outputDir) {
    
        String baseReleaseApkName = baseApk.name;
        channelList.each { channel ->
            String apkChannelName = getChannelApkName(baseReleaseApkName, channel)
            File destFile = new File(outputDir, apkChannelName)
            copyTo(baseApk, destFile)
            ChannelWriter.addChannelByV1(destFile, channel)
            if (!mRebuildChannelExtension.isFastMode) {
                //1. verify channel info
                if (ChannelReader.verifyChannelByV1(destFile, channel)) {
                    println("generateV1ChannelApk , ${destFile} add channel success")
                } else {
                    throw new GradleException("generateV1ChannelApk , ${destFile} add channel failure")
                }
                //2. verify v1 signature
                if (VerifyApk.verifyV1Signature(destFile)) {
                    println "generateV1ChannelApk , after add channel , apk : ${destFile} v1 verify success"
                } else {
                    throw new GradleException("generateV1ChannelApk , after add channel , apk : ${destFile} v1 verify failure")
                }
            }
        }

    }
```

* 这里可以看到方法里的逻辑就是拷贝一份到目的地址，然后ChannelWriter用V1签名的策略添加渠道信息
* 然后会根据是否定义了FastMode去校验渠道信息和V1签名的状况，如果设置了FastMode就不去检查这些信息了

## [ChannelWriter.addChannelByV1]

```java
   public static void addChannelByV1(File apkFile, String channel) throws Exception {
        V1SchemeUtil.writeChannel(apkFile, channel);
    }
```

## [V1SchemeUtil-> writeChannel]

```java
public static void writeChannel(File file, String channel) throws Exception {
    
        RandomAccessFile raf = null;
        byte[] comment = channel.getBytes(ChannelConstants.CONTENT_CHARSET);
        Pair<ByteBuffer, Long> eocdAndOffsetInFile = getEocd(file);
        if (eocdAndOffsetInFile.getFirst().remaining() == ZipUtils.ZIP_EOCD_REC_MIN_SIZE) {
          
            try {
                raf = new RandomAccessFile(file, "rw");
                //1.locate comment length field
                raf.seek(file.length() - ChannelConstants.SHORT_LENGTH);
                //2.write zip comment length (content field length + length field length + magic field length)
                writeShort(comment.length + ChannelConstants.SHORT_LENGTH + ChannelConstants.V1_MAGIC.length, raf);
                //3.write content
                raf.write(comment);
                //4.write content length
                writeShort(comment.length, raf);
                //5. write magic bytes
                raf.write(ChannelConstants.V1_MAGIC);
            } finally {
                if (raf != null) {
                    raf.close();
                }
            }
        } else {
           
            if (containV1Magic(file)) {
                try {
                    String existChannel = readChannel(file);
                    if (existChannel != null) {
                        file.delete();
                        throw new ChannelExistException("file : " + file.getAbsolutePath() + " has a channel : " + existChannel + ", only ignore");
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            int existCommentLength = ZipUtils.getUnsignedInt16(eocdAndOffsetInFile.getFirst(), ZipUtils.ZIP_EOCD_REC_MIN_SIZE - ChannelConstants.SHORT_LENGTH);
            int newCommentLength = existCommentLength + comment.length + ChannelConstants.SHORT_LENGTH + ChannelConstants.V1_MAGIC.length;
            try {
                raf = new RandomAccessFile(file, "rw");
                //1.locate comment length field
                raf.seek(eocdAndOffsetInFile.getSecond() + ZipUtils.ZIP_EOCD_REC_MIN_SIZE - ChannelConstants.SHORT_LENGTH);
                //2.write zip comment length (existCommentLength + content field length + length field length + magic field length)
                writeShort(newCommentLength, raf);
                //3.locate where channel should begin
                raf.seek(eocdAndOffsetInFile.getSecond() + ZipUtils.ZIP_EOCD_REC_MIN_SIZE + existCommentLength);
                //4.write content
                raf.write(comment);
                //5.write content length
                writeShort(comment.length, raf);
                //6.write magic bytes
                raf.write(ChannelConstants.V1_MAGIC);
            } finally {
                if (raf != null) {
                    raf.close();
                }
            }
        }
    }
```

* 这里的主要逻辑就是分为2大类，EOCD有commend内容和EOCD没有commend内容
* 如果原来的EOCD的长度就等于没有comment时的22，那么就读到EOCD的第20个字节的short也就是comment size去写入长度，长度是由**comment+comment_length+magic**
* 写入byte[] comment
* 写入short byte[] 的长度 
* 写入一个magic校验字段**{0x6c, 0x74, 0x6c, 0x6f, 0x76, 0x65, 0x7a, 0x68}**
* 如果EOCD不等于22，那就说明原本就是有commend信息，然后就会去校验commend信息里面有没有magic，就能证明这个comment是不是有VasDolly写入的，这里也揭示了为什么要把magic字段写在最后，这也是为了读取校验的时候比较舒服
* 如果从上一步检查出channel的信息存在的话那就把它删掉并抛出异常
* 否则就从EOCD中拿到comment长度并加上**comment+comment_length+magic**
* 然后在原来的comment后面在写入渠道信息
* 梳理完方法流程之后，看看这个写入方法中读取的方法怎么实现的

## [V1SchemeUtil-> readChannel]

```java
public static String readChannel(File file) throws Exception {
        RandomAccessFile raf = null;
        try {
            raf = new RandomAccessFile(file, "r");
            long index = raf.length();
            byte[] buffer = new byte[ChannelConstants.V1_MAGIC.length];
            index -= ChannelConstants.V1_MAGIC.length;
            raf.seek(index);
            raf.readFully(buffer);
            // whether magic bytes matched
            if (isV1MagicMatch(buffer)) {
                index -= ChannelConstants.SHORT_LENGTH;
                raf.seek(index);
                // read channel length field
                int length = readShort(raf);
                if (length > 0) {
                    index -= length;
                    raf.seek(index);
                    // read channel bytes
                    byte[] bytesComment = new byte[length];
                    raf.readFully(bytesComment);
                    return new String(bytesComment, ChannelConstants.CONTENT_CHARSET);
                } else {
                    throw new Exception("zip channel info not found");
                }
            } else {
                throw new Exception("zip v1 magic not found");
            }
        } finally {
            if (raf != null) {
                raf.close();
            }
        }
    }
```

* 首先创建一个magic长度的byte数组用于后面校验magic
* 然后就读出一个short型长度
* 然后在读出一个byte[] comment

## 阶段小结

来到这里，整个V1签名的渠道信息写入和读取的操作就分析完了，和Walle不同，VasDolly是采用在apk的EOCD区域添加comment信息实现渠道信息的写入

## 提出质疑

这里整个读写V1渠道信息的流程和步骤已经很清晰了，但是有一个东西是还不清晰的，那就是**ApkChannelPackageTask** 相比于 **RebuildApkChannelPackageTask**多了点什么？让它可以支持直接编译打包，而剩下的V2签名的渠道信息写入肯定就是一样的了

## [ApkChannelPackageTask-> channel]

```java
@TaskAction
    public void channel() {
        //1.check all params
        checkParameter();
        //2.check signingConfig , determine channel package mode
        checkSigningConfig()
        //3.generate channel apk
        generateChannelApk();
    }

```

发现没什么不同的地方，除了多了个检查SigningConfig之外，所以只能进一步退回到Plugin

## [ApkChannelPackagePlugin-> apply]

```java
project.afterEvaluate {
            project.android.applicationVariants.all { variant ->
                def variantOutput = variant.outputs.first();
                def dirName = variant.dirName;
                def variantName = variant.name.capitalize();
                Task channelTask = project.task("channel${variantName}", type: ApkChannelPackageTask) {
                    mVariant = variant;
                    mChannelExtension = mChannelConfigurationExtension;
                    mOutputDir = new File(mChannelConfigurationExtension.baseOutputDir, dirName)
                    isMergeExtensionChannelList = !mProject.hasProperty(PROPERTY_CHANNELS)
                    channelList = mChanneInfolList
                    dependsOn variant.assemble
                }
            }
        }

        project.task("reBuildChannel", type: RebuildApkChannelPackageTask) {
            isMergeExtensionChannelList = !mProject.hasProperty(PROPERTY_CHANNELS)
            channelList = mChanneInfolList
            mRebuildChannelExtension = mRebuildChannelConfigurationExtension
        }
```

* 发现其实就是构建task的时候不一样而已
* 继续V2签名的读写操作解析

## [RebuildApkChannelPackageTask-> generateV2ChannelApk]

```java
 void generateV2ChannelApk(File baseApk, File outputDir) {
      
        String baseReleaseApkName = baseApk.name;
        ApkSectionInfo apkSectionInfo = IdValueWriter.getApkSectionInfo(baseApk, mRebuildChannelExtension.lowMemory)
        channelList.each { channel ->
            String apkChannelName = getChannelApkName(baseReleaseApkName, channel)
            File destFile = new File(outputDir, apkChannelName)
            if (apkSectionInfo.lowMemory) {
                copyTo(baseApk, destFile)
            }
            ChannelWriter.addChannelByV2(apkSectionInfo, destFile, channel)
            if (!mRebuildChannelExtension.isFastMode) {
                //1. verify channel info
                if (ChannelReader.verifyChannelByV2(destFile, channel)) {
                    println("generateV2ChannelApk , ${destFile} add channel success")
                } else {
                    throw new GradleException("generateV2ChannelApk , ${destFile} add channel failure")
                }
                //2. verify v2 signature
                boolean success = VerifyApk.verifyV2Signature(destFile)
                if (success) {
                    println "generateV2ChannelApk , after add channel , apk : ${destFile} v2 verify success"
                } else {
                    throw new GradleException("generateV2ChannelApk , after add channel , apk : ${destFile} v2 verify failure")
                }
            }
            apkSectionInfo.rewind()
            if (!mRebuildChannelExtension.isFastMode) {
                apkSectionInfo.checkEocdCentralDirOffset()
            }
        }
    }
```

* 逻辑和V1签名添加逻辑大同小异
* 关注一下**lowMemory**这里的处理就是把baseApk拷贝到destFile中
* 而且这里的FastMode过滤的操作还包括了一个检查CentralDir的操作

## [ChannelWriter-> addChannelByV2]

```java
 public static void addChannelByV2(ApkSectionInfo apkSectionInfo, File destApk, String channel) throws IOException, ApkSignatureSchemeV2Verifier.SignatureNotFoundException {

        byte[] buffer = channel.getBytes(ChannelConstants.CONTENT_CHARSET);
        ByteBuffer channelByteBuffer = ByteBuffer.wrap(buffer);
        //apk中所有字节都是小端模式
        channelByteBuffer.order(ByteOrder.LITTLE_ENDIAN);

        IdValueWriter.addIdValue(apkSectionInfo, destApk, ChannelConstants.CHANNEL_BLOCK_ID, channelByteBuffer);
    }
```

## [IdValueWriter-> addIdValue]

```java
public static void addIdValue(ApkSectionInfo apkSectionInfo, File destApk, int id, ByteBuffer valueBuffer) throws IOException, ApkSignatureSchemeV2Verifier.SignatureNotFoundException {
      
        Map<Integer, ByteBuffer> idValueMap = new LinkedHashMap<>();
        idValueMap.put(id, valueBuffer);
        addIdValueByteBufferMap(apkSectionInfo, destApk, idValueMap);
    }
```

## [IdValueWriter-> addIdValueByteBufferMap]

```java
public static void addIdValueByteBufferMap(ApkSectionInfo apkSectionInfo, File destApk, Map<Integer, ByteBuffer> idValueMap) throws IOException, ApkSignatureSchemeV2Verifier.SignatureNotFoundException {
    
        Map<Integer, ByteBuffer> existentIdValueMap = V2SchemeUtil.getAllIdValue(apkSectionInfo.schemeV2Block.getFirst());
      existentIdValueMap.putAll(idValueMap);

        ByteBuffer newApkSigningBlock = V2SchemeUtil.generateApkSigningBlock(existentIdValueMap);
        
        ByteBuffer centralDir = apkSectionInfo.centralDir.getFirst();
        ByteBuffer eocd = apkSectionInfo.eocd.getFirst();
        long centralDirOffset = apkSectionInfo.centralDir.getSecond();
        int apkChangeSize = newApkSigningBlock.remaining() - apkSectionInfo.schemeV2Block.getFirst().remaining();
        //update the offset of centralDir
        ZipUtils.setZipEocdCentralDirectoryOffset(eocd, centralDirOffset + apkChangeSize);//修改了EOCD中保存的中央目录偏移量

        long apkLength = apkSectionInfo.apkSize + apkChangeSize;
        RandomAccessFile fIn = null;
        try {
            fIn = new RandomAccessFile(destApk, "rw");
            if (apkSectionInfo.lowMemory) {
                fIn.seek(apkSectionInfo.schemeV2Block.getSecond());
            } else {
                ByteBuffer contentEntry = apkSectionInfo.contentEntry.getFirst();
                fIn.seek(apkSectionInfo.contentEntry.getSecond());
                //1. write real content Entry block
                fIn.write(contentEntry.array(), contentEntry.arrayOffset() + contentEntry.position(), contentEntry.remaining());
            }

            //2. write new apk v2 scheme block
            fIn.write(newApkSigningBlock.array(), newApkSigningBlock.arrayOffset() + newApkSigningBlock.position(), newApkSigningBlock.remaining());
            //3. write central dir block
            fIn.write(centralDir.array(), centralDir.arrayOffset() + centralDir.position(), centralDir.remaining());
            //4. write eocd block
            fIn.write(eocd.array(), eocd.arrayOffset() + eocd.position(), eocd.remaining());
            //5. modify the length of apk file
            if (fIn.getFilePointer() != apkLength) {
                throw new RuntimeException("after addIdValueByteBufferMap , file size wrong , FilePointer : " + fIn.getFilePointer() + ", apkLength : " + apkLength);
            }
            fIn.setLength(apkLength);
        } finally {
            //恢复EOCD中保存的中央目录偏移量，满足基础包的APK结构
            ZipUtils.setZipEocdCentralDirectoryOffset(eocd, centralDirOffset);
            if (fIn != null) {
                fIn.close();
            }
        }
    }
```

* 首先第一步就是把原本的渠道信息读出来，然后把id-values putAll进去
* 然后就修改了EOCD中Central Directory的偏移量
* 接下来的操作就是分步写入，首先写入的content entries区域，所以这里揭示之前为什么要对lowMemory进行判断 ，如果是lowMemory的话就直接把整个整个apk拷贝过去然后在部分写入覆盖
* 下一步是写入Signing Block
* 写入Central dir
* 写入EOCD
* 校验长度
* 恢复EOCD中保存的中央目录偏移量

## [V2SchemeUtil-> getAllIdValue]

```JAVA
public static Map<Integer, ByteBuffer> getAllIdValue(ByteBuffer apkSchemeBlock) throws ApkSignatureSchemeV2Verifier.SignatureNotFoundException {
        ApkSignatureSchemeV2Verifier.checkByteOrderLittleEndian(apkSchemeBlock);
        // FORMAT:
        // OFFSET       DATA TYPE  DESCRIPTION
        // * @+0  bytes uint64:    size in bytes (excluding this field)
        // * @+8  bytes pairs
        // * @-24 bytes uint64:    size in bytes (same as the one above)
        // * @-16 bytes uint128:   magic
        ByteBuffer pairs = ApkSignatureSchemeV2Verifier.sliceFromTo(apkSchemeBlock, 8, apkSchemeBlock.capacity() - 24);
        Map<Integer, ByteBuffer> idValues = new LinkedHashMap<Integer, ByteBuffer>(); // keep order
        int entryCount = 0;
        while (pairs.hasRemaining()) {
            entryCount++;
            int len = (int) lenLong;
            int nextEntryPos = pairs.position() + len;
            int id = pairs.getInt();
            idValues.put(id, ApkSignatureSchemeV2Verifier.getByteBuffer(pairs, len - 4));//4 is length of id
            }
            pairs.position(nextEntryPos);
        }
        return idValues;
    }
```

* 整个读取的操作其实和**Walle**的实现大体一致，前8后24读取出pairs，然后就把读出来的数据放到LinkedHashMap，保证顺序的返回

