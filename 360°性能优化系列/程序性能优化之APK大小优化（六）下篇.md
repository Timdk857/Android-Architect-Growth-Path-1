**阿里P7移动互联网架构师进阶视频（每日更新中）免费学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**

本篇文章将继续从微信资源混淆AndResGuard原理来介绍APK大小优化:
微信的AndResGuard工具是用于Android资源的混淆，作用有两点：一是通过混淆资源ID长度同时利用7z深度压缩，减小了apk包大小；二是混淆后在安全性方面有一点提升，提高了逆向破解难度。本文从源码角度，来探寻AndResGuard实现原理。

阅读本文需要前提知识：掌握Android应用程序打包编译过程，尤其是对资源的编译和打包过程；熟悉resource.arsc文件格式。

推荐罗升阳文章：http://blog.csdn.net/luoshengyang/article/details/8744683 
微信资源混淆工具源码地址：https://github.com/shwenzhang/AndResGuard 
附上来自网络神图：
![](https://upload-images.jianshu.io/upload_images/19956127-5ebe6cb98b1d0dfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**0、程序入口CliMain.main()**
该函数处理命令行参数、并解析自定义配置文件，混淆工具可以根据配置项进行特定处理，具体参考config.xml内容，针对其中特定内容，我们会在后面提到。然后进入真正混淆的入口函数resourceProgurad()

特别说明一下解析Configuration中关键点，处理复用旧的mapping文件： 
**1、processOldMappingFile()**
```
private void processOldMappingFile() throws IOException {
        ...
        try {
            String line = br.readLine();

            while (line != null) {
                if (line.length() > 0) {
                    Matcher mat = MAP_PATTERN.matcher(line);

                    if (mat.find()) {
                        String nameAfter = mat.group(2);
                        String nameBefore = mat.group(1);
                        nameAfter = nameAfter.trim();
                        nameBefore = nameBefore.trim();

                        //如果有这个的话，那就是mOldFileMapping
                        if (line.contains("/")) {
                            mOldFileMapping.put(nameBefore, nameAfter);
                        } else {
                            //这里是resid的mapping
                            int packagePos = nameBefore.indexOf(".R.");
                            if (packagePos == -1) {
                                throw new IOException(
                                    String.format(
                                        "the old mapping file packagename is malformed, " +
                                            "it should be like com.tencent.mm.R.attr.test, yours %s\n", nameBefore)
                                );

                            }
                            String packageName = nameBefore.substring(0, packagePos);
                            int nextDot = nameBefore.indexOf(".", packagePos + 3);
                            String typeName = nameBefore.substring(packagePos + 3, nextDot);

                            String beforename = nameBefore.substring(nextDot + 1);
                            String aftername = nameAfter.substring(nameAfter.indexOf(".", packagePos + 3) + 1);

                            HashMap<String, HashMap<String, String>> typeMap;

                            if (mOldResMapping.containsKey(packageName)) {
                                typeMap = mOldResMapping.get(packageName);
                            } else {
                                typeMap = new HashMap<>();
                            }

                            HashMap<String, String> namesMap;
                            if (typeMap.containsKey(typeName)) {
                                namesMap = typeMap.get(typeName);
                            } else {
                                namesMap = new HashMap<>();
                            }
                            namesMap.put(beforename, aftername);

                            typeMap.put(typeName, namesMap);
                            mOldResMapping.put(packageName, typeMap);
                        }
                    }

                }
                line = br.readLine();
            }
        }
        ...
    }
}
```
该函数主要功能是：对oldmapping文件处理是按照正则表达式把“->”分隔提取两边字符串，进行hashmap缓存:

其一、如果有这个“/”的话，那就是res path mapping即mOldFileMapping的hashmap中： 
mOldFileMapping.put(nameBefore, nameAfter); 
(例如res/drawable -> r/c，最终mOldFileMapping是(“res/drawable”,”r/c”))

其二、否则判断如果包含“.R.”,则是resid的mapping，最后按照类别、package保存到oldResMapping的hashmap中： 
namesMap.put(beforename, aftername); 
(例如com.basket24.demo.R.attr.progress -> com.basket24.demo.R.attr.a，最终namesMap是(“progress”,”a”)) 
typeMap.put(typeName, namesMap); 
(例如com.basket24.demo.R.attr.progress -> com.basket24.demo.R.attr.a，最终typeMap是(“attr”,namesMap)) 
mOldResMapping.put(packageName, typeMap); 
(例如com.basket24.demo.R.attr.progress -> com.basket24.demo.R.attr.a，最终mOldResMapping是(“com.basket24.demo”,typeMap))

**2、Main.resourceProguard()是混淆真正的入口。**
```
protected void resourceProguard(File outputFile, String apkFilePath, InputParam.SignatureType signatureType) {
        ApkDecoder decoder = new ApkDecoder(config);
        File apkFile = new File(apkFilePath);
        ...
        mRawApkSize = FileOperation.getFileSizes(apkFile);
        try {
            /* 默认使用V1签名 */
            decodeResource(outputFile, decoder, apkFile);
            buildApk(decoder, apkFile, signatureType);
        } catch (Exception e) {
            e.printStackTrace();
            goToError();
        }
    }
```
混淆入口resourceProguard里功能： 
其一：decodeResource();//进行混淆资源相关功能。 
其二：buildApk(decoder, apkFile, signatureType);//最后buildApk生成签名包。

3、Main.decodeResource()
```
private void decodeResource(File outputFile, ApkDecoder decoder, File apkFile) {
    decoder.setApkFile(apkFile);
    ...
    decoder.setOutDir(mOutDir.getAbsoluteFile());
    decoder.decode();//混淆资源功能
}
```
decodeResource核心功能就是设置相关变量，并执行ApkDecoder.decode()。

**4、ApkDecoder.decode()**
```
public void decode(){
    if (hasResources()) {
        ensureFilePath();       
        RawARSCDecoder.decode(mApkFile.getDirectory().getFileInput("resources.arsc"));
        ResPackage[] pkgs = ARSCDecoder.decode(mApkFile.getDirectory().getFileInput("resources.arsc"), this);

        //把没有纪录在resources.arsc的资源文件也拷进dest目录
        copyOtherResFiles();

        /*把整个arsc重新修改其中几个字符串池和对应大小，形成新的arsc文件。*/
        ARSCDecoder.write(mApkFile.getDirectory().getFileInput("resources.arsc"), this, pkgs);
      }
}
```
**5、ensureFilePath()**
```
ensureFilePath(){
    Utils.cleanDir(mOutDir);//mOutDir就是outapk目录

    //temp目录，用于解压apk
    String unZipDest = new File(mOutDir, TypedValue.UNZIP_FILE_PATH).getAbsolutePath();

    mCompressData = FileOperation.unZipAPk(mApkFile.getAbsoluteFile().getAbsolutePath(), unZipDest);
    dealWithCompressConfig();//
    //将res混淆成r
    if (!config.mKeepRoot) {
        mOutResFile = new File(mOutDir.getAbsolutePath() + File.separator + TypedValue.RES_FILE_PATH);
    } else {
        mOutResFile = new File(mOutDir.getAbsolutePath() + File.separator + "res");
    }

    //这个需要混淆各个文件夹
    // TypedValue.UNZIP_FILE_PATH指"temp"
    mRawResFile = new File(mOutDir.getAbsoluteFile().getAbsolutePath() + File.separator + TypedValue.UNZIP_FILE_PATH + File.separator + "res");
    mOutTempDir = new File(mOutDir.getAbsoluteFile().getAbsolutePath() + File.separator + TypedValue.UNZIP_FILE_PATH);

    //这里遍历获取原始res目录的文件
    Files.walkFileTree(mRawResFile.toPath(), new ResourceFilesVisitor());

    mOutTempARSCFile = new File(mOutDir.getAbsoluteFile().getAbsolutePath() + File.separator + "resources_temp.arsc");
    mOutARSCFile = new File(mOutDir.getAbsoluteFile().getAbsolutePath() + File.separator + "resources.arsc");

    String basename = mApkFile.getName().substring(0, mApkFile.getName().indexOf(".apk"));

    //RES_MAPPING_FILE = "resource_mapping_";
    //mResMappingFile名称如“resource_mapping_imfun.txt"
    mResMappingFile = new File(mOutDir.getAbsoluteFile().getAbsolutePath() + File.separator
        + TypedValue.RES_MAPPING_FILE + basename + TypedValue.TXT_FILE);
}
```
ensureFilePath主要功能如下： 
其一、在输出目录下，建立一个temp目录，用于apk解压的目录。unZipAPk解压apk，得到mCompressData压缩条目集合[compress.put(name,entry.getMethod());] 
其二、根据config来修改压缩的值，将满足config的压缩类型，进行修改压缩标记为ZIP_DEFLATED 
其三、判断是否将将res混淆成r 
其四、创建需要混淆的temp目录(apk被解压到temp目录)等、使用FileVisitor对目录进行遍历，将原始res（”temp/res”）下路径保存到HashSet中。 
其五、创建resources_temp.arsc 和最终resources.arsc等文件及最终mapping命名：resource_mapping_apkname.txt

下面回到第4步ApkDecoder.decode()中继续执行：

**6、RawARSCDecoder.decode()** 
这一步就是解析原始resources.arsc文件，得到文件结构并缓存相关数据，如资源类型字符串池mExistTypeNames等。代码较长，且关键步骤较少，故略去代码。

继续在第4步ApkDecoder.decode()中往下执行：

**7、ARSCDecoder.decode()**
```
public static ResPackage[] decode(InputStream arscStream, ApkDecoder apkDecoder){
    try {
         //proguardFileName混淆文件名
         ARSCDecoder decoder = new ARSCDecoder(arscStream, apkDecoder);
         ResPackage[] pkgs = decoder.readTable();
         return pkgs;
     } catch (IOException ex) {
         throw new AndrolibException("Could not decode arsc file", ex);
     }
}
```
**8、ARSCDecoder的构造函数中执行proguardFileName()**
```
proguardFileName(){

    //其中初始化ProguardStringBuilder，建立各种被映射为的字符集合标记集合
    mProguardBuilder = new ProguardStringBuilder();
    mProguardBuilder.reset();

    final Configuration config = mApkDecoder.getConfig();
    File rawResFile = mApkDecoder.getRawResFile();
    File[] resFiles = rawResFile.listFiles();
    if (!config.mKeepRoot) {
        //需要保持之前的命名方式
        if (config.mUseKeepMapping) {
            mOldFileMapping提取values部分即"r/c"保存到keepFileNames，然后从mProguardBuilder生成的混淆字符池中删除掉这些names.
            for (File resFile : resFiles) {
                String raw = "res" + "/" + resFile.getName();
                if (fileMapping.containsKey(raw)) {
                    mOldFileName.put(raw, fileMapping.get(raw));
                } else {
                    mOldFileName.put(raw, resRoot + "/" + mProguardBuilder.getReplaceString());
                }
            }
            /*上面mOldFileName保存的是用旧混淆(没有的话从新的混淆池中获取)文件处理过的File混淆映射.
            （mOldFileName("res/attr"," r/h")）*/
        }else{//否则
            for (int i = 0; i < resFiles.length; i++) {
                //这里也要用linux的分隔符,如果普通的话，就是r
                mOldFileName.put("res" + "/" + resFiles[i].getName(), TypedValue.RES_FILE_PATH + "/" + mProguardBuilder.getReplaceString());
            }
        }

        generalFileResMapping();//资源目录File映射
    }
}

/**
*对资源目录File映射。
*/
generalFileResMapping(){
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
这里第8步主要功能是： 
其一、其中初始化ProguardStringBuilder，建立混淆字符串池和标记集合。 
其二、获取配置config内容，判断是否keeproot,是否沿用旧的mapping文件等，进行映射。 
其三、generalFileResMapping把缓存的映射hashmap写入文件，形成mapping文件，其中目前只有资源目录path映射。

回到第7步中继续执行decoder.readTable()进行真正混淆

**9、decoder.readTable()**
```
ResPackage[] readTable(){
    mTableStrings = StringBlock.read(mIn);
    ResPackage[] packages = new ResPackage[packageCount];
    packages[i] = readPackage();
    return packages;
}
```
readPackage()解析resources.arsc文件，其中关键步骤readEntry()如下：

**10、readEntry()**
```
readEntry(){
    if (config.mUseWhiteList) {
        //判断是否走whitelist
        HashMap<String, HashMap<String, HashSet<Pattern>>> whiteList = config.mWhiteList;
        String packName = mPkg.getName();
        if (whiteList.containsKey(packName)) {

            HashMap<String, HashSet<Pattern>> typeMaps = whiteList.get(packName);
            String typeName = mType.getName();

            if (typeMaps.containsKey(typeName)) {
                String specName = mSpecNames.get(specNamesId).toString();
                HashSet<Pattern> patterns = typeMaps.get(typeName);
                for (Iterator<Pattern> it = patterns.iterator(); it.hasNext(); ) {
                    Pattern p = it.next();
                    if (p.matcher(specName).matches()) {
                        mPkg.putSpecNamesReplace(mResId, specName);//缓存设置package中spec替换项
                        mPkg.putSpecNamesblock(specName);
                        mProguardBuilder.setInWhiteList(mCurEntryID, true);//当前资源项ID标示为白名单

                        mType.putSpecProguardName(specName);//设置spec的proguard的名称为原始资源项名称
                        isWhiteList = true;
                        break;
                    }
                }
            }

        }


    }


    if (!isWhiteList) {
        boolean keepMapping = false;
        if (config.mUseKeepMapping) {//判断旧的mapping文件也复用，得到replaceString
            HashMap<String, HashMap<String, HashMap<String, String>>> resMapping = config.mOldResMapping;
            String packName = mPkg.getName();
            //resMapping是指res Id的映射
            if (resMapping.containsKey(packName)) {
                HashMap<String, HashMap<String, String>> typeMaps = resMapping.get(packName);
                String typeName = mType.getName();
                if (typeMaps.containsKey(typeName)) {
                    //这里面的东东已经提前去掉，请放心使用
                    /*(例如com.basket24.demo.R.attr.progress -> com.basket24.demo.R.attr.a，最终proguard是("progress","a"))*/
                    HashMap<String, String> proguard = typeMaps.get(typeName);
                    String specName = mSpecNames.get(specNamesId).toString();
                    if (proguard.containsKey(specName)) {
                        keepMapping = true;
                        /*获取旧的混淆id映射中specname对应的混淆字符串，继续使用。*/
                        replaceString = proguard.get(specName);
                    }
                }
            }
        }

        //没有经过旧的混淆文件处理，则直接从混淆池中获取一个混淆字符串
        if (!keepMapping) {
            replaceString = mProguardBuilder.getReplaceString();
        }

        /*设置混淆池中对应资源项id的位置为“已混淆”的标记。*/
        mProguardBuilder.setInReplaceList(mCurEntryID, true);
        if (replaceString == null) {
            throw new AndrolibException("readEntry replaceString == null");
        }
        //根据新的混淆字符串，生成相应的id映射。
        generalResIDMapping(mPkg.getName(), mType.getName(), mSpecNames.get(specNamesId).toString(), replaceString);

        //以下对混淆字符串进行相应对象的缓存。
        mPkg.putSpecNamesReplace(mResId, replaceString);
        mPkg.putSpecNamesblock(replaceString);
        mType.putSpecProguardName(replaceString);
    }
}  

/*根据新的混淆字符串，生成相应的id映射。输出到新的混淆mapping文件中(里面已经文件file的映射关系)。*/
generalResIDMapping(){
    mMappingWriter.write("    " + packagename + ".R." + typename + "." + specname + " -> " + packagename + ".R." + typename + "." + replace);
}
```
readEntry函数主要实现了： 
其一、判断是否启用whitelist，如果有的话，设置specname的混淆字符串为原始字符串，即不进行混淆，进行相应对象缓存。 
其二、判断是否复用旧的mapping文件中id的映射，已有的继续使用旧的映射关系中的混淆字符串，否则从混淆池中获取一个新的字符串，即得到replaceString。 
其三、根据新的混淆字符串，生成相应的id映射。输出到新的混淆mapping文件中(里面已经文件file的映射关系)。

readEntry继续解析arsc文件，执行到关键步骤readValue:

**11、readValue()**
```
readValue() {
    //这里面有几个限制，一对于string ,id, array我们是知道肯定不用改的，第二看要那个type是否对应有文件路径
    if (mPkg.isCanProguard() && flags && type == TypedValue.TYPE_STRING && mShouldProguardForType && mShouldProguardTypeSet.contains(mType.getName())) {
        //mTableStringsProguard是要存放混淆的资源项值
        if (mTableStringsProguard.get(data) == null) {
            String raw = mTableStrings.get(data).toString();//mTableStrings是解析原始arsc文件得到资源项值字符串池
            String proguard = mPkg.getSpecRepplace(mResId);//获取前面已缓存下的specName对应的混淆字符串
            //这个要写死这个，因为resources.arsc里面就是用这个"/"
            int secondSlash = raw.lastIndexOf("/");
            ...
            String newFilePath = raw.substring(0, secondSlash);//获得原始资源项值的path部分

            if (!mApkDecoder.getConfig().mKeepRoot) {
                //如在(“res/drawable“,”r/c”)中找到newFilePath=”r/c”
                newFilePath = mOldFileName.get(raw.substring(0, secondSlash));//mOldFileName是已生成的混淆文件映射
            }
            ...
            //同理这里不能用File.separator，因为resources.arsc里面就是用这个

            /***********************
            *结果result如”r/c/a”
            ************************/
            String result = newFilePath + "/" + proguard; 
            ...
            String compatibaleraw = new String(raw);
            String compatibaleresult = new String(result);

            //为了适配window要做一次转换
            if (!File.separator.contains("/")) {
                compatibaleresult = compatibaleresult.replace("/", File.separator);
                compatibaleraw = compatibaleraw.replace("/", File.separator);
            }

            //下面很关键，创建了原始res文件和混淆后的res文件
            File resRawFile = new File(mApkDecoder.getOutTempDir().getAbsolutePath() + File.separator + compatibaleraw);
            File resDestFile = new File(mApkDecoder.getOutDir().getAbsolutePath() + File.separator + compatibaleresult);

            //这里用的是linux的分隔符
            HashMap<String, Integer> compressData = mApkDecoder.getCompressData();
            if (compressData.containsKey(raw)) {
                compressData.put(result, compressData.get(raw));//替换压缩的文件名为混淆后的字符串
            } else {
                System.err.printf("can not find the compress dataresFile=%s\n", raw);
            }

            if (!resRawFile.exists()) {
                System.err.printf("can not find res file, you delete it? path: resFile=%s\n", resRawFile.getAbsolutePath());
                return;
            } else {
                if (resDestFile.exists()) {
                    throw new AndrolibException(
                        String.format("res dest file is already  found: destFile=%s", resDestFile.getAbsolutePath())
                    );
                }
                /**************************************************************
                *关键点：把旧的资源文件内容copy到新的混淆后的资源文件中
                **************************************************************/
                FileOperation.copyFileUsingStream(resRawFile, resDestFile);
                //already copied
                //从原始资源目录mRawResourceFiles中删除掉该已混淆的文件Path
                mApkDecoder.removeCopiedResFile(resRawFile.toPath());

                /**********************
                *按照data的index顺序，保存resutl(result如”r/c/a”),
                *即把混淆后的资源项的值缓存下来
                **********************/
                mTableStringsProguard.put(data, result);
            }
        }
    }
}
```
readValue主要实现了： 
其一、mPkg.getSpecRepplace获取前面已缓存下的specName对应的混淆字符串如“a” 
其二、从mOldFileName中如在(“res/drawable“,”r/c”)中找到newFilePath=”r/c” 
其三、生成result如”r/c/a” 
其四、创建了混淆后的res文件，把旧的资源文件内容copy到新的混淆后的资源文件中。 
其五、从原始资源目录mRawResourceFiles中删除掉该已混淆的文件Path 
其六、按照Value的index顺序，保存result(如”r/c/a”),即把混淆后的资源项的值缓存下来

下面回到第4步中，继续执行copyOtherResFiles():

**12、 copyOtherResFiles()**
```
copyOtherResFiles(){
    ...
    Path resPath = mRawResFile.toPath();
    Path destPath = mOutResFile.toPath();

    //mRawResourceFiles中是剩下的
    for (Path path : mRawResourceFiles) {
        //copy文件内容到dest中
        FileOperation.copyFileUsingStream(path.toFile(), dest.toFile());

    }
}
```
该函数主要实现了把没有纪录在resources.arsc的资源文件也拷进dest目录。

回到第4步中，继续执行ARSCDecoder.write()：

**13、ARSCDecoder.write()**
```
write(){
    ARSCDecoder writer = new ARSCDecoder(arscStream, decoder, pkgs);
    writer.writeTable();
}

writeTable(){
    System.out.printf("writing new resources.arsc \n");
    mTableLenghtChange = 0;
    writeNextChunkCheck(Header.TYPE_TABLE, 0);
    int packageCount = mIn.readInt();
    mOut.writeInt(packageCount);

    //mTableStringsProguard就是上面产生的已混淆的资源项值的字符串池
    mTableLenghtChange += StringBlock.writeTableNameStringBlock(mIn, mOut, mTableStringsProguard);
    ...
    for (int i = 0; i < packageCount; i++) {
        mCurPackageID = i;
        writePackage();
    }
    //最后需要把整个的size重写回去
    reWriteTable();
}


writePackage(){
    checkChunkType(Header.TYPE_PACKAGE);
    int id = (byte) mIn.readInt();
    mOut.writeInt(id);
    mResId = id << 24;
    //char_16的，一共256byte
    mOut.writeBytes(mIn, 256);
    /* typeNameStrings */
    mOut.writeInt(mIn.readInt());
    /* typeNameCount */
    mOut.writeInt(mIn.readInt());
    /* specNameStrings */
    mOut.writeInt(mIn.readInt());
    /* specNameCount */
    mOut.writeInt(mIn.readInt());
    StringBlock.writeAll(mIn, mOut);

    if (mPkgs[mCurPackageID].isCanProguard()) {
        //writeSpecNameStringBlock把混淆后specname重新写入arsc文件
        //其中mCurSpecNameToPos是混淆的specname对应位置
        int specSizeChange = StringBlock.writeSpecNameStringBlock(
            mIn,
            mOut,
            mPkgs[mCurPackageID].getSpecNamesBlock(),
            mCurSpecNameToPos
        );
        mPkgsLenghtChange[mCurPackageID] += specSizeChange;
        mTableLenghtChange += specSizeChange;//重新记录大小
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

/**
*修改混淆资源项specname对应位置
*/
writeEntry(){
        /* size */
        mOut.writeBytes(mIn, 2);
        short flags = mIn.readShort();
        mOut.writeShort(flags);
        int specNamesId = mIn.readInt();
        ResPackage pkg = mPkgs[mCurPackageID];
        if (pkg.isCanProguard()) {

            //获取资源项specname对应位置
            specNamesId = mCurSpecNameToPos.get(pkg.getSpecRepplace(mResId));
            if (specNamesId < 0) {
                throw new AndrolibException(String.format(
                    "writeEntry new specNamesId < 0 %d", specNamesId));
            }
        }
        //重写位置
        mOut.writeInt(specNamesId);

        if ((flags & ENTRY_FLAG_COMPLEX) == 0) {
            writeValue();
        } else {
            writeComplexEntry();
        }
    }
```
这一步同样是解析resource.arsc,重新修改arsc文件其中几个字符串池和对应大小，形成新的arsc文件。主要包括： 
其一、资源项值字符串池修改，我们需要把文件指向路径改变，例如res/layout/test.xml,改为res/layout/a.xml 
其二、资源项key池修改，即specsname除了白名单部分全部废弃，替换成所有我们混淆方案中用到的字符。 
其三、每个资源项entry中指向的specsname中的id修正。由于specname已混淆，我们需要用混淆后的资源项specname的位置改写。

回到最开始第2步中，执行 buildApk(decoder, apkFile, signatureType);

**14、buildApk()** 
重新打包生成新的apk并签名等，这一步不再赘述。

以上完成了对apk资源混淆的过程分析。

# 总结：

资源混淆核心处理过程如下： 
**1、生成新的资源文件目录，里面对资源文件路径进行混淆(其中涉及如何复用旧的mapping文件)，例如将res/drawable/hello.png混淆为r/s/a.png，并将映射关系输出到mapping文件中。** 
**2、对资源id进行混淆(其中涉及如何复用旧的mapping文件)，并将映射关系输出到mapping文件中。** 
**3、生成新的resources.arsc文件，里面对资源项值字符串池、资源项key字符串池、进行混淆替换，对资源项entry中引用的资源项字符串池位置进行修正、并更改相应大小，并打包生成新的apk。**

原文链接https://blog.csdn.net/cg_wang/article/details/70183864
**阿里P7移动互联网架构师进阶视频（每日更新中）免费学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**
