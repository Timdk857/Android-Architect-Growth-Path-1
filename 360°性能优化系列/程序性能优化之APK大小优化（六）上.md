**阿里P7移动互联网架构师进阶视频（每日更新中）免费学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**

本篇文章将继续从APK瘦身来介绍APK大小优化:
文章主要内容从理论出发，再做实际操作。分为下面几个方面：1. 结构分析， 2.具体实操 3. 总结 
# 1. 结构分析
首先上传一张瘦身前通过Analyze app分析出来的图片（打开方式：Android Studio下 ——> Build——> Analyze app）：
![](https://upload-images.jianshu.io/upload_images/19956127-1f8a84efe2db8ebb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**APK包结构如下：**
lib/：包含特定于处理器软件层的编译代码。该目录包含了每种平台的子目录，像armeabi，armeabi-v7a， arm64-v8a，x86，x86_64，和mips。大多数情况下我们可以只用一种armeabi-v7a，后面会讲到原因。
assets/：包含应用可以使用AssetManager对象检索的应用资源。
res/：包含未编译到的资源 resources.arsc,主要有图片资源文件。
META-INF/：包含CERT.SF和 CERT.RSA签名文件以及MANIFEST.MF 清单文件。
resources.arsc：包含已编译的资源。该文件包含res/values/ 文件夹所有配置中的XML内容。打包工具提取此XML内容，将其编译为二进制格式，并将内容归档。此内容包括语言字符串和样式，以及直接包含在resources.arsc文件中的内容路径 ，例如布局文件和图像。
classes.dex：包含以Dalvik / ART虚拟机可理解的DEX文件格式编译的类。
AndroidManifest.xml：包含核心Android清单文件。该文件列出应用程序的名称，版本，访问权限和引用的库文件。该文件使用Android的二进制XML格式。
通过分析图可以知道，目前app主要是so文件占比比较大，占了31.7M,占了整个应用是38.2%。其次是assets目录，整个目录占了32M,第三就是资源文件res目录了。所以接下来我们处理步骤就是按这个顺序来处理。（简单说下图中的Raw File Size（磁盘解压后的大小）和DownLoad Size（从应用商店下载的大小），如果想了解更多关于Analyaer分析的知识，可以参考这篇文章使用APK Analyzer分析你的APK)，分析了包结构组成之后，我们可以开始瘦身操作了。

## 2.具体实操
#### 2.1. 对lib目录下的文件进行瘦身处理
**2.1.1 修改lib配置：**
参考资料
so文件的优化：通常我们在使用NDK开发的时候，我们经常会有如下这么一段代码:
```
ndk {
            //设置支持的so库架构
            abiFilters "armeabi-v7a", "x86", "arm64-v8a", "x86_64", "armeabi"
        }
```
最后我的修改代码如下：
```
ndk 	{
            //设置支持的so库架构
            abiFilters "armeabi-v7a"
        }
```
接下来说明这么做的依据：
看上面图分析，armeabi-v7主要不支持ARMv5(1998年诞生)和ARMv6(2001年诞生).目前这两款处理器的手机设备基本不在我公司的适配范围（市场占比太少）。
而许多基于 x86 的设备也可运行 armeabi-v7a 和 armeabi NDK 二进制文件。对于这些设备，主要 ABI 将是 x86，辅助 ABI 是 armeabi-v7a。
最后总结一点：如果适配版本高于4.1版本，可以直接像我上面这样写，当然，如果armeabi-v7a不是设备主要ABI，那么会在性能上造成一定的影响。
参考文章：安卓app打包的时候还需要兼容armeabi么？

好了，我们再打一次包试试。
![](https://upload-images.jianshu.io/upload_images/19956127-30b2a87a38f4d892.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

确实有点震惊，一下子包小了这么多，从87.1M到51.9M,容我好好算算少了多少M.赶快让测试帮忙测一下。基于之前的理论知识，心里还是有点底。果然，测试效果和之前是一样的。心里的石头先落下罗。

**2.1.2 重新编译so文件，用更小的库代替**
相信很多开发者都有这种苦恼，很多第三方我们导入进来只用到其中很小一部分功能，大部分功能都是我们用不上的。这时候我们找到源代码，将我们需要的那部分代码提取出来，重新编译成新的so文件，再导入到我们项目中。当然，如果之前没有编译过so文件，这部分建议做最后的优化去处理。不然你会遇到很多问题。上一波处理后的效果图：
![](https://upload-images.jianshu.io/upload_images/19956127-1ed0f553172a7740.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里说下，因为项目中有使用到ffmpeg库，之前导入的第三方的放在assets文件夹下，重写编写后的so库文件放在lib文件夹下，所以lib文件夹反而大了。从51.9M到35.6M,效果还是蛮不错的。

对了，别问我为什么assets文件夹下为什么还有12.6M资源，因为很多.mp3都是第三方的人脸识别必备配置文件，我也很无奈。
![](https://upload-images.jianshu.io/upload_images/19956127-608b8eb3fce8e0c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 2.2  优化res,assets文件大小
**1. 手动lint检查，手动删除无用资源**
在Android Studio中打开“Analyze” 然后选择"Inspect Code…"，范围选择整个项目，然后点击"OK"。配置如下：

![](https://upload-images.jianshu.io/upload_images/19956127-7449b7d5460ce8a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2. 使用tinypng等图片压缩工具对图片进行压缩。**
打开网址，将大图片导入到tinypng，替换之前的图片资源。

**3. 大部分图片使用Webp格式代替。**
可以给UI提要求，让他们将图片资源设置为Webp格式，这样的话图片资源会小很多。如果想了解更多关于webp,请点击这里webp，当然，如果对图片颜色通道要求不高，可以考虑转jpg,最好用webp,因为效果更佳。

**4. 尽量不要在项目中使用帧动画**
一个帧动画几十张图片，再怎么压缩都还是占很大内存比重的。所以建议是让UI去搞，这里可以参考使用lottie-android，如果项目中动画效果多的话效果更加明显。当然这就要辛苦我们UI设计师大大了。

**5. 使用gradle开启shrinkResources**
移除无用资源文件，下面是我的配置：
```
 buildTypes {
        release {
            // 不显示Log
            buildConfigField "boolean", "LOG_DEBUG", "false"
            //混淆
            minifyEnabled true
            // 移除无用的resource文件
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
    }
```
通过上述步骤操作，apk效果如下:
![](https://upload-images.jianshu.io/upload_images/19956127-61884e0e94e5687e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

又优化了将近5M,别问我为什么还有7.5M，里面大量的gif和webp格式的动图，都是UI丢给我的，一个2.7M.后面再慢慢和他细究这个问题。后面要做的两部分，一部分是将资源文件下的所有gif图放后台下载处理，第二个是和UI讨论下如何减小webp 动图的大小（我看其他平台只有100K的样子，给我的就2.7M?）。

#### 2.3. 减少chasses.dex大小
classes.dex中包含了所有的java代码，当你打包时，gradle会将所有模板力的.class文件转换成classes.dex文件，当然，如果方法数超过64K，将要新增其他文件进行存储。可以通过multidexing分多个文件，比如我这里的chasses2.dex。换句话说，就是减少代码量。我们可以通过以下方法来实现：

尽量减少第三方库的引用，这个在上面我们已经做过优化了。
避免使用枚举，这里特别去网上查了一下，具体可以参考下这篇文章Android 中的 Enum 到底占多少内存？该如何用？，得出的结论是，可能几十个枚举的内存占有量才相当一张图片这样子，优化效果也不会特别明显。当然，如果你是个追求极致的人，我不反对你用静态常量替代枚举。
如果你的dex文件太大，检查是否引入了重复功能的第三方库（图片加载库，glide,picasso,fresco,image_loader，如果不是你一个人单独开发完成的很容易出现这种情况），尽量做到一个功能点一个库解决。
关于classes.dex文件大小分析可以参考这篇译文使用 APK Analyzer 分析你的 APK

#### 2.4. 其他
删除无用的语7zip代替
删除翻译资源，只保留中英文
尝试将andorid support库彻底踢出你的项目。
尝试使用动态加载so库文件，插件化开发。
将大资源文件放到服务端，启动后自动下载使用。
# 3. 总结
好了，说道这里基本上就结束了，apk包从87.1M减小到了23.1M（优化了73%,不要说我标题党）已经差不多了，关于第四部其他部分的优化我是没有进行再操作的。因为公司运营觉得二三十M的包比较真实，太小了就太假了。所以我暂时就不进行优化了。如果再上面提到的部分通过所有将所有非启动页面首页之外的所有资源，so库放服务端，理论上apk包大小能在10M以内这样子。当然我们有做到就不多加评价了。最后，如果对插件化开发感兴趣的话可以参考下这篇文章Android全面插件化方案-RePlugin踩坑。
原文链接https://blog.csdn.net/qq_32175491/article/details/80071987#4__133
**阿里P7移动互联网架构师进阶视频（每日更新中）免费学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**
