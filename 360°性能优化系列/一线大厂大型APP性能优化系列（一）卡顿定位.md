## 0.序

作者将近7年Android开发，经历过很多一线公司的APP开发，如中石油，阿里，京东等，想把真正一些一线的APP里的优秀的经验分享出来，打算利用休息时间更新一个系列的《APP性能优化》，大约是20章节，每周大约会更新2章，喜欢的朋友记得加个关注和点赞。如有笔误欢迎指出。

讲解的内容大体包含，异步优化，启动优化，卡顿优化，内存优化，ARTHook, 监控耗时盲区，网络，电量，瘦身及APP容灾方案等

## 1.简介

本篇文章是该系列文章中的第一篇，主要介绍的是在一些一线大厂的实际项目中，如果APP发生卡顿是如何进行定位问题的。主要介绍 程序的耗费时间

## 2.测量时间方式

首先，如果要查看页面加载花费的时间有3种方式

1.  adb命令查看
2.  手动打点的方式
3.  traceView

## 3.adb命令

只需要一行命令，就可以查看加载页面的时间。

```
adb shell am start -W 包名/包名.Activity
```

使用后会显示

![](https://upload-images.jianshu.io/upload_images/22976303-94548c5d56677db3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ThisTime: 代表启动最后一个Activity的耗时
TotalTime: 代表启动所有的Activity的耗时
WaitTime: 代表AMS启动所有的Activity的耗时

> 注意点：该命令只能是获取配置了的Activity, 其他的无效，因为Android组件中有个 exported 属性，没有intent-filter时exported 属性默认为false，此组件只能由本应用用户访问，配备了intent-filter后此值改变为true，允许外部调用。

**缺点**： adb命令只能查看配置了的Activity，其他的无法查看，并且也无法精准的查看其方法具体耗费的时间，所以因为其局限性 并不能很好的解决我们问题，我们采用手动打点的方式查看。

## 4.手动打点

先看代码

```
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        initBugly();

        initBaiduMap();

        initJPushInterface();

        initShareSDK();
        ...
    }

    private void initBugly() throws InterruptedException {

        Thread.sleep(1000); // 模拟耗费的时间
    }

    private void initBaiduMap()  throws InterruptedException {

        Thread.sleep(2000); // 模拟耗费的时间
    }

    private void initJPushInterface() throws InterruptedException {

        Thread.sleep(3000); // 模拟耗费的时间
    }

    private void initShareSDK() throws InterruptedException {

        Thread.sleep(500); // 模拟耗费的时间
    }
}
复制代码
```

代码不用我说，项目中很常见，现在的问题是APP启动加载很慢, 那么如何精准的查询到具体耗时的方法？

```
  long startTime = System.currentTimeMillis();
  initBugly();
  Log.d("lybj", "initBugly()方法耗时："+ (System.currentTimeMillis() - startTime));

  long startTime = System.currentTimeMillis();
  initBaiduMap();
  Log.d("lybj", "initBaiduMap()方法耗时："+ (System.currentTimeMillis() - startTime));
  ...
复制代码
```

这样可以吗？当然不行，耦合性太大，每一个方法都加，那么测试完了，删除代码也是个体力活，一不小心删错一个，就会造成灾难性的问题，在实际项目中，比如中石油的一些项目，就会采用 AOP 的方式来测量方法的耗费时长。

### 4.1 AOP

> AOP : Aspect Oriented Programming的缩写，意为：面向切面编程

优点：

1.  针对同一问题的统一处理
2.  无侵入添加代码

这里我们使用的是Aspectj

### 4.2 Aspectj 的使用

#### 1.添加依赖

根目录的build.gradle里

```
buildscript {
    ...
    dependencies {
        ...
        classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.0'
    }
}
复制代码
```

app项目的build.gradle及新建的module的build.gradle里添加

```
apply plugin: 'android-aspectjx'

dependencies {
    ...
    implementation 'org.aspectj:aspectjrt:1.8.+'
}
复制代码
```

#### 2.创建切面

```
@Aspect
public class PerformanceAop {

    @Around("call(* com.bj.performance.MyApplication.**(..))")
    public void getTime(ProceedingJoinPoint joinPoint){

        long startTime = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().getName();
        try {
            joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        Log.d("lybj", methodName + "方法耗时："+ (System.currentTimeMillis() - startTime));
    }
}
复制代码
```

看，根本无需修改任何工程代码，就可以获取运行时长了，点击运行显示

![](https://upload-images.jianshu.io/upload_images/22976303-4a6b08d8c5cb3e69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[AspectJ语法参考](https://www.jianshu.com/p/f90e04bcb326)

**缺点**： 如果项目比较庞大，上百个方法，总不能全部打点，然后一个一个的分析到底是哪个地方运行时间过长吧，所以我们需要一个比较直观的工具，一眼就能看到具体哪个方法运行时间过长。

## 5\. traceView的使用

### 5.1 特点

1.  图形的形式展示其执行时间调用栈
2.  信息全面，包含所有进程

### 5.2 使用方式

```
Debug.startMethodTracing("文件名");

Debug.stopMethodTracing();
复制代码
```

在代码中相应位置的地方打入埋点即可， startMethodTracing 有3个构造参数分别是

1.  tracePath：文件名/路径
2.  bufferSize：文件的容量大小
3.  flag：TRACE_COUNT_ALLOCS 只有默认的这一种

代码运行完成后，会在

> mnt/sdcard/Android/data/包名/files

生成一个.trace后缀的文件，可以用Profiler添加打开它。

> 当然也可以使用Profiler的录制功能，但是因为要测量启动时间，点击录制手速并不会那么的精准，所以采用埋点的方式获取trace文件进行分析。

### 5.3 性能分析

![](https://upload-images.jianshu.io/upload_images/22976303-fc4e5d6a21d9f1ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开文件后为上图所示

### 5.4 时间模式

上图标签 1 所示：wallclock time 和 cpu time

1.  Wall Clock Time：从进程开始运行到结束，时钟走过的时间，这其中包含了进程在阻塞和等待状态的时间。
2.  Thread Time ：就是CPU执行用户指令所用的时间。

> 注意：如果线程A执行函数b，但是因为函数b加了锁，线程A进入等待状态，那么wallclock time也是要计算时间的，而Thread time则只是计算CPU花费在它身上的时间。
> 
> 一般根据经验来讲wall duration时间久说明耗时多，然后看Thread time 这个值说明cpu花费在其身上的时间多不多。不多的话基本可以直接异步（因为不抢占CPU）多的话需要合理调度好执行顺序。

### 5.5 Call Chart

上图标签 2 所示：Call Chart

![](https://upload-images.jianshu.io/upload_images/22976303-bbcf25477ea42fd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

它的作用就是可以很直观的查看到底是哪里耗时比较久，x轴为调用的时间线，越宽的表示耗时越久，y轴为调用的深度，也就是调用的子方法。父类在最上面，很明显initBottomTab（）方法调用是最耗时的。

> 鼠标悬浮可以查看耗费时间，双击可以跳转到相应代码

*   橙色：系统方法
*   蓝色：第三方API（包括java语言的api）
*   绿色：App自身方法

简易图如下：

![](https://upload-images.jianshu.io/upload_images/22976303-3a1922a81a5889ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5.6 Flame Chart

上图标签 3 所示：Flame Chart 又称之为火焰图

![](https://upload-images.jianshu.io/upload_images/22976303-f49265c443c8288b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。 x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的调用栈合并后，按字母顺序排列的。

> 火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"（plateaus），就表示该函数可能存在性能问题。

#### 练习1：

![](https://upload-images.jianshu.io/upload_images/22976303-465405cce7676cbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

假如火焰图如上图所示，我们需要分析哪个函数吗？

> 答：最顶层的函数g()占用 CPU 时间最多。d()的宽度最大，但是它直接耗用 CPU 的部分很少。b()和c()没有直接消耗 CPU。因此，如果要调查性能问题，首先应该调查g()，其次是i()。 另外，从图中可知a()有两个分支b()和h()，这表明a()里面可能有一个条件语句，而b()分支消耗的 CPU 大大高于h()。

#### 与Call Chart区别

方法D对B(B1、B2和B3)进行多次调用，其中一些调用B对C(C1和C3)进行调用。如果用Call Chart表示，则为

![](https://upload-images.jianshu.io/upload_images/22976303-2563e87aa79cb8d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为B1、B2和B3共享相同的序列调用者(A→D→B)聚合,如下所示。同样,C1和C3聚合,因为它们共享相同的序列调用者(A→D→B→C)注意不包括C2, 因为它有不同的调用者序列(A→D→C)。

![](https://upload-images.jianshu.io/upload_images/22976303-6e2481dda80277e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么如果使用火焰图，则表示：

![](https://upload-images.jianshu.io/upload_images/22976303-5fbc4963fab5317a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是说，收集相同的调用序列的相同方法被收集并表示为火焰图中的一个较长的栏(而不是将它们显示为多个更短的条）

### 5.7 Top Down

上图标签 4 所示： Top Down 显示一个函数调用列表，在该列表中展开函数节点会显示函数的被调用方

![](https://upload-images.jianshu.io/upload_images/22976303-2468ff51b921779a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看上图右边区域：

*   Self:方法调用用于执行自己的代码而不是它的子类的时间量。
*   Children：方法调用花费的时间用于执行其被调用者，而不是其自己的代码
*   Total：方法的Self和Children的时间的总和。这表示应用程序执行方法调用的总时间量

### 5.8 Bottom Up

上图标签 5 所示： Bottom Up 显示一个函数调用列表，在该列表中展开函数节点将显示函数的调用方。

![](https://upload-images.jianshu.io/upload_images/22976303-d8d53c3165fd1635.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Bottom Up选项卡对于那些消耗最多(或最少)CPU时间的方法的排序方法很有用。可以检查每个节点，以确定哪些调用者在调用这些方法上花费最多的CPU时间。

*   Self:方法调用用于执行自己的代码而不是它的子类的时间量。
*   Children：方法调用花费的时间用于执行其被调用者，而不是其自己的代码
*   Total：方法的Self和Children的时间的总和。这表示应用程序执行方法调用的总和

简略图如下

![](https://upload-images.jianshu.io/upload_images/22976303-8dbf6c49c0c97734.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**缺点**： 在项目中用到的，最多的还是Call Chart 和 Top Down, 但是traceView的原理就是抓取所有线程的所有函数里的信息，所以会导致程序变慢， 所以常用的是SysTrace

SysTrace 作者用的不多，有兴趣的朋友可以自行百度，用法和traceView的使用差不多，就不在分析了

转自作者：刘洋巴金

