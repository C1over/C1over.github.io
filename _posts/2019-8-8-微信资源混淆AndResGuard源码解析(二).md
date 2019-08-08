---
layout:     post   				    
title:      微信资源混淆AndResGuard源码解析(二) 
subtitle:   apk学习系列   #副标题
date:       2019-8-8		   	# 时间
author:     Cc1over				# 作者
header-img: img/post-bg-map.jpg              #这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - apk学习系列
---

# 微信资源混淆AndResGuard源码解析(二)

## 前言

 上一篇主要是针对整个resource.arsc文件的解析作分析，而这一篇除了上一篇的忽略的两个点之外，更多就是写回操作的分析

## [ApkDecoder-> decode]

```java
public void decode() throws AndrolibException, IOException, DirectoryException {
    if (hasResources()) {
      ensureFilePath();
      // read the resources.arsc checking for STORED vs DEFLATE compression
      // this will determine whether we compress on rebuild or not.
      System.out.printf("decoding resources.arsc\n");
      RawARSCDecoder.decode(apkFile.getDirectory().getFileInput("resources.arsc"));
      ResPackage[] pkgs = ARSCDecoder.decode(apkFile.getDirectory().getFileInput("resources.arsc"), this);

      //把没有纪录在resources.arsc的资源文件也拷进dest目录
      copyOtherResFiles();

      ARSCDecoder.write(apkFile.getDirectory().getFileInput("resources.arsc"), this, pkgs);
    }
  }
```

## [ARSCDecoder-> write]

```java
public static void write(InputStream arscStream, ApkDecoder decoder, ResPackage[] pkgs) throws AndrolibException {
    try {
      ARSCDecoder writer = new ARSCDecoder(arscStream, decoder, pkgs);
      writer.writeTable();
    } catch (IOException ex) {
      throw new AndrolibException("Could not decode arsc file", ex);
    }
  }
```

## [ARSCDecoder-> writeTable]

```java
private void writeTable() throws IOException, AndrolibException {
    
    mTableLenghtChange = 0;
    writeNextChunkCheck(Header.TYPE_TABLE, 0);
    int packageCount = mIn.readInt();
    mOut.writeInt(packageCount);
    // 1
    mTableLenghtChange += StringBlock.writeTableNameStringBlock(mIn, mOut, mTableStringsResguard);
    writeNextChunk(0);
    
    for (int i = 0; i < packageCount; i++) {
      mCurPackageID = i;
      writePackage();
    }
    // 最后需要把整个的size重写回去
    reWriteTable();
  }
```

* 这个写入操作对于一些table的数据边读边写的策略
* 注释1：带着解析操作填充好的**mTableStringsResguard**这个map去执行全局常量池的写入操作
* 执行packages的写入操作
* 执行table的size写入和拷贝过程
* 这里的逻辑思路大概就是因为，做完资源混淆之后，table的长度变化了，而现在解析完的这个阶段，这些映射关系是存在内存中的，直接用内存中的map的关系计算出变化的长度不是很合适，所以就创建一个临时文件，先把混淆后的结果写到这个临时文件中，然后边写边计算出table的变化长度，然后，再把长度信息，和临时文件里的信息写回到正式文件中

## [StringBlock-> writeTableNameStringBlock]

```java
public static int writeTableNameStringBlock(
      ExtDataInput reader, ExtDataOutput out, Map<Integer, String> tableProguardMap)
      throws IOException, AndrolibException {
    int type = reader.readInt();
    int chunkSize = reader.readInt();
    int stringCount = reader.readInt();
    int styleOffsetCount = reader.readInt();
    int flags = reader.readInt();
    int stringsOffset = reader.readInt();
    int stylesOffset = reader.readInt();

    StringBlock block = new StringBlock();
    block.m_isUTF8 = (flags & UTF8_FLAG) != 0;
    if (block.m_isUTF8) {
      System.out.printf("resources.arsc Character Encoding: utf-8\n");
    } else {
      System.out.printf("resources.arsc Character Encoding: utf-16\n");
    }

    block.m_stringOffsets = reader.readIntArray(stringCount);
    block.m_stringOwns = new int[stringCount];
    for (int i = 0; i < stringCount; i++) {
      block.m_stringOwns[i] = -1;
    }
    if (styleOffsetCount != 0) {
      block.m_styleOffsets = reader.readIntArray(styleOffsetCount);
    }
}
```

* 首先就是把全局常量池中的从chunk header开始到style这部分的信息，读取出来，但是这里埋下的一个伏笔就是stringOwns中的项先置为了-1

```java
{
      int size = ((stylesOffset == 0) ? chunkSize : stylesOffset) - stringsOffset;
      if ((size % 4) != 0) {
        throw new IOException("String data size is not multiple of 4 (" + size + ").");
      }
      block.m_strings = new byte[size];
      reader.readFully(block.m_strings);
    }
```

* 这里的逻辑就是计算出strings的长度，因为要考虑到由styles和没有styles的情况，所以计算规则则是由styles偏移量不为0即styles是存在的，那么直接把styles的偏移量减去strings的偏移量就是strings的长度了，如果没有那么就以为着strings的结尾就是整个块的结尾了，所以直接用chunksize减就可以了
* 然后这里有一个判断就是这个strings的长度必须是4的倍数
* 然后就是填充这个strings数组

```java
if (stylesOffset != 0) {
      int size = (chunkSize - stylesOffset);
      if ((size % 4) != 0) {
        throw new IOException("Style data size is not multiple of 4 (" + size + ").");
      }
      block.m_styles = reader.readIntArray(size / 4);
    }
```

* 这里的逻辑也类似，就是当styles存在的时候，styles的长度也必须是4的倍数

```java
  int totalSize = 0;
  out.writeCheckInt(type, CHUNK_STRINGPOOL_TYPE);
  totalSize += 4;
   
  totalSize += 6 * 4 + 4 * stringCount + 4 * styleOffsetCount;
  stringsOffset = totalSize;
  byte[] strings = new byte[block.m_strings.length];
  int[] stringOffsets = new int[stringCount];
  System.arraycopy(block.m_stringOffsets, 0, stringOffsets, 0, stringOffsets.length);
```

* 这里就是一路计算到strings的偏移量的位置，然后重新创建一个strings和stringsOffsets数组，然后先把原来的stringsOffsets拷贝过去
* 这个totalSize后面可能还会用上

```java
    int offset = 0;
    int i;
    for (i = 0; i < stringCount; i++) {
      stringOffsets[i] = offset;
      //如果找不到即没混淆这一项,直接拷贝
      if (tableProguardMap.get(i) == null) {
        //需要区分是否是最后一项
        int copyLen = (i == (stringCount - 1)) ? (block.m_strings.length - block.m_stringOffsets[i])
            : (block.m_stringOffsets[i + 1] - block.m_stringOffsets[i]);
        System.arraycopy(block.m_strings, block.m_stringOffsets[i], strings, offset, copyLen);
        offset += copyLen;
        totalSize += copyLen;
      }
```

* 这里描述的就是在**tableProguardMap**找不到对应项的情况，就直接拷贝原来的字符串，这里有一个操作就是区分最后一项

```java
       String name = tableProguardMap.get(i);
        if (block.m_isUTF8) {
          strings[offset++] = (byte) name.length();
          strings[offset++] = (byte) name.length();
          totalSize += 2;
          byte[] tempByte = name.getBytes(Charset.forName("UTF-8"));
          // 1  
          if (name.length() != tempByte.length) {
            throw new AndrolibException(String.format(
                "writeTableNameStringBlock UTF-8 length is different  name %d, tempByte %d\n",
                name.length(),
                tempByte.length
            ));
          }
          System.arraycopy(tempByte, 0, strings, offset, tempByte.length);
          offset += name.length();
          strings[offset++] = NULL;
          totalSize += name.length() + 1;
        } else {
          writeShort(strings, offset, (short) name.length());
          offset += 2;
          totalSize += 2;
          byte[] tempByte = name.getBytes(Charset.forName("UTF-16LE"));
          if ((name.length() * 2) != tempByte.length) {
            throw new AndrolibException(String.format(
                "writeTableNameStringBlock UTF-16LE length is different  name %d, tempByte %d\n",
                name.length(),
                tempByte.length
            ));
          }
          System.arraycopy(tempByte, 0, strings, offset, tempByte.length);
          offset += tempByte.length;
          strings[offset++] = NULL;
          strings[offset++] = NULL;
          totalSize += tempByte.length + 2;
        }
      }
```

* 这段逻辑描述的就是**tableProguardMap**命中的情况
* 首先就是对编码格式的兼容，这里判断长度的原因主要是因为utf-16编码是以2个字节编码的，所以理所当然的解码的过程也是按照2个字节去进行解析，而utf-8则比较特殊，它要根据额外的标识信息去判断长度，所以就意味着utf-8是采用1-3去表示原来的编码的，所以根据长度的判断就可以判定编码是否一致
* 校验完之后的操作就是array copy过去，然后接着往上加total size
* 然后如果它本来就不是用utf-8编码方式的话，那么同样道理校验一下是否为2的倍数，然后不是就抛异常，是的话就执行array copy以及total size的往上添加

```java
//要保证string size 是4的倍数,要补零
    int size = totalSize - stringsOffset;
    if ((size % 4) != 0) {
      int add = 4 - (size % 4);
      for (i = 0; i < add; i++) {
        strings[offset++] = NULL;
        totalSize++;
      }
    }
```

* 然后这一段就是为了保证混淆修改了字符串之后整个strings的长度还是4的倍数

```java
//因为是int的,如果之前的不为0
    if (stylesOffset != 0) {
      stylesOffset = totalSize;
      totalSize += block.m_styles.length * 4;
    }

    out.writeInt(totalSize);
    out.writeInt(stringCount);
    out.writeInt(styleOffsetCount);
    out.writeInt(flags);
    out.writeInt(stringsOffset);
    out.writeInt(stylesOffset);
    out.writeIntArray(stringOffsets);
    if (stylesOffset != 0) {
      out.writeIntArray(block.m_styleOffsets);
    }
    out.write(strings, 0, offset);
    if (stylesOffset != 0) {
      out.writeIntArray(block.m_styles);
    }
    return (chunkSize - totalSize);
```

* 然后的操作就是常规的，写入，然后返回一个全局常量池中长度差值

## 阶段小结

整个全局常量池的写入操作的本质就是把之前在**readValue**阶段保存的**全局常量池索引值**和**资源名称**得到的一个映射表，转换一个新的strings字节数组，然后就写入进去，这个阶段的易错点有两个，一个是编码格式，另一个就是4的倍数这个问题，编码格式如果不做强校验的话，写进去了之后反编译再看的话就会乱码，而4的倍数问题更严重，如果这里不做强对齐的话，就会导致apk安装不了！

## [ARSCDecoder-> writeTable]

```java
   for (int i = 0; i < packageCount; i++) {
      mCurPackageID = i;
      writePackage();
    }
```

## [ARSCDecoder-> writePackage]

```java
private void writePackage() throws IOException, AndrolibException {
   
    mResId = id << 24;
    // 1
    StringBlock.writeAll(mIn, mOut);

    if (mPkgs[mCurPackageID].isCanResguard()) {
      int specSizeChange = StringBlock.writeSpecNameStringBlock(mIn,
         mOut,
         mPkgs[mCurPackageID].getSpecNamesBlock(),
         mCurSpecNameToPos
      );
      mPkgsLenghtChange[mCurPackageID] += specSizeChange;
      mTableLenghtChange += specSizeChange;
    } else {
      StringBlock.writeAll(mIn, mOut);
    }
    writeNextChunk(0);
    while (mHeader.type == Header.TYPE_LIBRARY) {
      writeLibraryType();
    }
    while (mHeader.type == Header.TYPE_SPEC_TYPE) {
      writeTableTypeSpec();
    }
  }
```

* 常规在mResId的高4位的记录下packageId
* 注释1：然后资源类型的stringBlock就不混淆直接写入
* 然后就判断一下当前拿到的这个包是否是可以混淆，这个值是在decode的时候就设置上去的，只有系统的包名是不允许混淆的
* 然后就写入nameSpec这个stringBlock，拿到变化的diff记录下来

## [StringBlock-> writeSpecNameStringBlock]

```java
if (styleOffsetCount != 0) {
      throw new AndrolibException(String.format("writeSpecNameStringBlock styleOffsetCount != 0  styleOffsetCount %d",
          styleOffsetCount
      ));
    }
```

* 强校验让specName的styles必须没有

```java
byte[] stringBytes = new byte[size * 2];
    int offset = 0;
    int i = 0;
 curSpecNameToPos.clear();
```

* 创建2倍size的strings数组

```java
for (Iterator<String> it = specNames.iterator(); it.hasNext(); ) {
      stringOffsets[i] = offset;
      String name = it.next();
      curSpecNameToPos.put(name, i);
      if (isUTF8) {
        stringBytes[offset++] = (byte) name.length();
        stringBytes[offset++] = (byte) name.length();
        totalSize += 2;
        byte[] tempByte = name.getBytes(Charset.forName("UTF-8"));
        if (name.length() != tempByte.length) {
          throw new AndrolibException(String.format(
              "writeSpecNameStringBlock %s UTF-8 length is different name %d, tempByte %d\n",
              name,
              name.length(),
              tempByte.length
          ));
        }
        System.arraycopy(tempByte, 0, stringBytes, offset, tempByte.length);
        offset += name.length();
        stringBytes[offset++] = NULL;
        totalSize += name.length() + 1;
      } else {
        writeShort(stringBytes, offset, (short) name.length());
        offset += 2;
        totalSize += 2;
        byte[] tempByte = name.getBytes(Charset.forName("UTF-16LE"));
        if ((name.length() * 2) != tempByte.length) {
          throw new AndrolibException(String.format(
              "writeSpecNameStringBlock %s UTF-16LE length is different name %d, tempByte %d\n",
              name,
              name.length(),
              tempByte.length
          ));
        }
        System.arraycopy(tempByte, 0, stringBytes, offset, tempByte.length);
        offset += tempByte.length;
        stringBytes[offset++] = NULL;
        stringBytes[offset++] = NULL;
        totalSize += tempByte.length + 2;
      }
      i++;
    }
```

* 这个写入的操作步骤和逻辑和之前全局常量池的写入是差不多的，不过不一样的是，这里传入了一个curSpecNameToPos用来记录specName和位置的映射表，这里猜测应该是后面entry或者value处理的时候要用到这个东西
* 这个curSpecNameToPos对应了ARSCDecoder中的**mCurSpecNameToPos**

## [ARSCDecoder-> writeTableTypeSpec]

```java
private void writeTableTypeSpec() throws AndrolibException, IOException {
    mResId = (0xff000000 & mResId) | id << 16;
    while (writeNextChunk(0).type == Header.TYPE_TYPE) {
      writeConfig();
    }
  }
```

* 例行地往mResId写入资源id

## [ARSCDecoder-> writeConfig]

```java
private void writeConfig() throws IOException, AndrolibException {
   
   for (int i = 0; i < entryOffsets.length; i++) {
      if (entryOffsets[i] != -1) {
        mResId = (mResId & 0xffff0000) | i;
        writeEntry();
      }
    }
  }
```

* 例行加入entryId，得到完整的资源id

## [ARSCDecoder-> writeEntry]

```java
 private void writeEntry() throws IOException, AndrolibException {
    ResPackage pkg = mPkgs[mCurPackageID];
    if (pkg.isCanResguard()) {
      specNamesId = mCurSpecNameToPos.get(pkg.getSpecRepplace(mResId));
    }
    mOut.writeInt(specNamesId);

    if ((flags & ENTRY_FLAG_COMPLEX) == 0) {
      writeValue();
    } else {
      writeComplexEntry();
    }
  }
```

* 符合上面的猜想，那就是**mCurSpecNameToPos**这个索引表是在entry写入中使用的，这里看到的感悟就是数据结构用得行云流水，因为在**readEntry**的过程中分别会保存**资源id**和**混淆名称/资源名称**的映射以及就是**混淆名称/资源名称**的一个集合Set，因为这个**混淆名称/资源名称**的容器肯定是要不能重复的，但是以此为代价就是无序，所以才需要把**资源id**和**混淆名称/资源名称**的映射关系的保存起来
* 因为在上一步写入specName的时候其实遍历set得到的混淆名称肯定是和一开始放进去的顺序不一样的，所以就手动通过**mCurSpecNameToPos**记录下set中拿到的值与offset的关系
* 然后通过计算出来的**资源id**就可以拿到当时在**readEntry**保存的名称，然后通过这个名称在从**mCurSpecNameToPos**里面拿到索引值，那么这个索引值就是**specNamesId**了

## [ARSCDeocder-> writeValue]

```java
private void writeValue() throws IOException, AndrolibException {
    /* size */
    mOut.writeCheckShort(mIn.readShort(), (short) 8);
    /* zero */
    mOut.writeCheckByte(mIn.readByte(), (byte) 0);
    byte type = mIn.readByte();
    mOut.writeByte(type);
    int data = mIn.readInt();
    mOut.writeInt(data);
  }
```

* 然后因为全局常量池是按照顺序一一写入的，所以就没有specName那里那种问题，所以**writeValue**就相对没有那么复杂

## 总结

整个写入的操作和之前猜想是一致的，那就是把读取过程中记录在内存的数据，写回到文件里面，写回分两步，先是写到临时文件，而且要计算混淆之后的长度，然后再把临时文件的内容写到真实文件，而且要写道混淆后的长度