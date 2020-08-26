**更多Android高级架构进阶视频学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**

# 1、对象优先在Eden分配
大多情况，对象在新生代Eden区分配。当Eden区没有足够空间进行分配时，虚拟机将进行一次Minor GC。虚拟机提供了参数 -XX:+PrintGCDetails ，在虚拟机发生垃圾收集行为时打印内存回收日志。

**新生代Minor GC 事例**

定义了4个字节数组对象，3个2MB大小、1个4MB大小，

通过-Xms20M -Xmx20M -Xmn10M 三个参数限制了Java堆大小为 20M ，不可扩展，其中的 10MB 分配给新生代，剩下 10MB 分配给老年代

-XX:SurvivorRatio=8 新生代 Eden 与 Survivor 区空间比例是 8:1:1
```
package com.lkf.jvm;

public class MinorGCDemo {
    private static final int _1MB = 1024 * 1024;

    /**
     * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
     * */
    public static void testAllocation() {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB]; /出现一次 Minor GC

    }

    public static void main(String[] args) {
        testAllocation();
    }
}
```
结果：
```
[GC (Allocation Failure) Disconnected from the target VM, address: '127.0.0.1:61454', transport: 'socket'
[PSYoungGen: 6570K->704K(9216K)] 6570K->4808K(19456K), 0.0036571 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 7253K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 8192K, 79% used [0x00000007bf600000,0x00000007bfc657a0,0x00000007bfe00000)
  from space 1024K, 68% used [0x00000007bfe00000,0x00000007bfeb0000,0x00000007bff00000)
  to   space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
 ParOldGen       total 10240K, used 4104K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  object space 10240K, 40% used [0x00000007bec00000,0x00000007bf002020,0x00000007bf600000)
 Metaspace       used 3104K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 392K, committed 512K, reserved 1048576K
```
从输出结果可以清晰看到 “eden space 8192K、from space 1024K、to space 1024K” 
新生代总可用空间为 9216KB (Eden区空间大小 + 1个Survivor区的总容量)

这次GC发生的原因是给allocation4对象分配内存的时候，发现Eden区已经被占用了6MB，剩余空间已经不足以分配4MB的内存，因此发生了MinorGC。GC期间有发现已有的3个2MB大小的对象已经无法全部放入Survivor空间（只有1MB大小）,所以只好通过分配担保机制提前将这三个对象转移到老年代去了。

# 2、大对象直接进入老年代
所谓大对象是指，需要大量连续内存空间的Java对象，经常出现大对象容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来为大对象分配内存。

虚拟机提供了一个-XX:PretenureSizeThreshold 参数，让大于该值得对象直接进入老年代。这样做的目的是避免在新生代Eden区及两个Survivor区之间发生大量的内存复制。

PretenureSieThreshold 参数只对 Serial 和 ParNew 两款收集器有效，Parallel Scavenge 收集器不识别这个参数，并且该收集器一般不需要设置。如果必须使用此参数的场合，可以考虑ParNew加CMS的收集器组合。

jdk1.7 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）

jdk1.8 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）

jdk1.9 默认垃圾收集器G1

-XX:+PrintCommandLineFlags 可查看默认设置收集器类型

-XX:+PrintGCDetails 打印的GC日志
```
package com.lkf.jvm;

public class PretenureSizeThresholdDemo {
    private static final int _1MB = 1024 * 1024;

    /**
     * VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
     * -XX:+PrintCommandLineFlags -XX:PretenureSizeThreshold=3145728 -XX:+UseSerialGC
     * 因为使用的是jdk1.8，所以此处特指定了使用垃圾收集器Serial
     * 大于3M的对象直接进入老年代
     */
    public static void testPretenureSizeThreshold() {
        byte[] allocation;
        allocation = new byte[4 * _1MB];//直接分配在老年代
    }

    public static void main(String[] args) {
        testPretenureSizeThreshold();
    }
}
```
结果：
```
Heap
 def new generation   total 9216K, used 2643K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  32% used [0x00000007bec00000, 0x00000007bee94ee8, 0x00000007bf400000)
  from space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
  to   space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
 tenured generation   total 10240K, used 4096K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
   the space 10240K,  40% used [0x00000007bf600000, 0x00000007bfa00010, 0x00000007bfa00200, 0x00000007c0000000)
 Metaspace       used 3104K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 392K, committed 512K, reserved 1048576K
```
# 3、长期存活的对象将进入老年代
虚拟机使用了分代收集的思想来管理内存，内存回收时为了区分哪些对象应放在新生代，哪些应该放在老年代，虚拟机为每个对象定义了一个对象年龄（Age）计数器。

如果对象被分配在Eden区并经过第一次Minor GC 后仍然存活，并且能被Survivor容乃的情况下，将被移动到Survivor中，对象年龄设为1。在Survivor区每经过一次Minor GC，年龄就加1，当对象的年龄到达一定程度时（默认15岁），就会晋升到老年代。对象晋升到老年代的阈值，可以通过参数：-XX:MaxTenuringThreshold 设置。
```
package com.lkf.jvm;

/**
 * 长期存活对象将进入老年代
 */
public class MaxTenuringThresholdDemo {
    private static final int _1MB = 1024 * 1024;

    /**
     * VM 参 数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -XX:+PrintTenuringDistribution
     * */
    public static void testTenuringThreshold() {
        byte[] allocation1, allocation2, allocation3;
        allocation1 = new byte[_1MB / 4];
        //什么时候进入老年代取决于XX:MaxTenuringThreshold设置
        allocation2 = new byte[4 * _1MB];
        allocation3 = new byte[4 * _1MB];
        allocation3 = null;
        allocation3 = new byte[4 * _1MB];
    }

    public static void main(String[] args) {
        testTenuringThreshold();
    }
}
```
-XX:MaxTenuringThreshold=1 （jdk1.8）运行结果：
```
Heap
 PSYoungGen      total 9216K, used 6994K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 8192K, 85% used [0x00000007bf600000,0x00000007bfcd4b68,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 ParOldGen       total 10240K, used 8192K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  object space 10240K, 80% used [0x00000007bec00000,0x00000007bf400020,0x00000007bf600000)
 Metaspace       used 3104K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 392K, committed 512K, reserved 1048576K
```
-XX:MaxTenuringThreshold=15 （jdk1.8）运行结果：
```
Heap
 PSYoungGen      total 9216K, used 6994K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 8192K, 85% used [0x00000007bf600000,0x00000007bfcd4b68,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 ParOldGen       total 10240K, used 8192K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  object space 10240K, 80% used [0x00000007bec00000,0x00000007bf400020,0x00000007bf600000)
 Metaspace       used 3104K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 392K, committed 512K, reserved 1048576K
```
# 4、动态对象年龄判定
虚拟机并不是永远要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果Survivor空间中，相同年龄对象的大小之和大于Survivor空间大小的一半，就可以直接进入老年代。
```
package com.lkf.jvm;

/**
 * 动态对象年龄判定
 */
public class TenuringThresholdDemo {
    private static final int _1MB = 1024 * 1024;

    /**
     * VM 参 数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15 -XX:+PrintTenuringDistribution -XX:+UseSerialGC
     * */
    public static void testTenuringThreshold() {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[_1MB / 4];
        allocation2 = new byte[_1MB / 4];
        //allocation1+allocation2大于Survivo空间的一半
        allocation3 = new byte[4 * _1MB];
        allocation4 = new byte[4 * _1MB];
        allocation4 = null;
        allocation4 = new byte[4 * _1MB];

    }

    public static void main(String[] args) {
        testTenuringThreshold();
    }
}
```
输出结果：
```
[GC (Allocation Failure) [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 15)
- age   1:    1048568 bytes,    1048568 total
: 7087K->1023K(9216K), 0.0048599 secs] 7087K->5179K(19456K), 0.0048867 secs] [Times: user=0.00 sys=0.01, real=0.01 secs] 
[GC (Allocation Failure) [DefNew
Desired survivor size 524288 bytes, new threshold 15 (max 15)
: 5283K->0K(9216K), 0.0018311 secs] 9439K->5148K(19456K), 0.0018528 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 4260K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  52% used [0x00000007bec00000, 0x00000007bf0290e0, 0x00000007bf400000)
  from space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
  to   space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
 tenured generation   total 10240K, used 5148K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
   the space 10240K,  50% used [0x00000007bf600000, 0x00000007bfb07198, 0x00000007bfb07200, 0x00000007c0000000)
 Metaspace       used 3104K, capacity 4568K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 392K, committed 512K, reserved 1048576K
```
# 5、空间分配担保
在发生Minor GC 之前，虚拟机会先检查老年代最大可用连续空间是否大于新生代所有对象大小总和，如果条件成立，那么Minor GC可以确保是安全的。如果不成立，虚拟机会查看HandlePromotionFailure设置的值是否允许担保失败。如果允许，那么虚拟机会检查老年代最大可用连续空间是否大于历次晋升到老年代对象大小的平均值，如果大于，将会尝试进行一次Minor GC；如果小于，或者HandlePromotionFailure设置不允许冒险，这时会进行一次Full GC。

JDK 6 Update 24 之后，HandlePromotionFailure参数不会再影响到虚拟机空间分配担保的策略，规则变为只要老年代的连续空间大于新生代对象总大小或者大于历次晋升对象大小的平均值就会进行Minor GC ，否则将进行Full GC。

Minor GC 和 Full GC的区别

* 新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为Java对象大多都有朝生夕死的特性，所以Minor GC 很频繁，一般回收速度也很快。

* 老年代GC（Major GC/Full GC）:指发生在老年代的GC，出现了Major GC ，经常会伴随至少一次的Minor GC。Major GC 的速度一般会比Minor GC 慢10倍。
**更多Android高级架构进阶视频学习请点击：[https://space.bilibili.com/474380680](https://links.jianshu.com/go?to=https%3A%2F%2Fspace.bilibili.com%2F474380680)**
原文链接：[https://www.cnblogs.com/liukaifeng/p/10052626.html](https://www.cnblogs.com/liukaifeng/p/10052626.html)
