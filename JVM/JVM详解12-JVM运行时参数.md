# JVM详解12-JVM运行时参数

# 1.JVM参数选项类型

## 1.1.类型一:标准参数选项

* 比较稳定,后续版本基本不会变化,以`-`开头;
* 运行`java -help`可以看到所有的标准选项;

## 1.2.类型二:-X参数选项

* 功能比较稳定,但官方说后续版本可能会变更,以`-X`开头;
* 运行`java -X`可以看到所有的X选项;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211109195449157.png" alt="image-20211109195449157" style="zoom: 50%;" />

## 1.2.类型三:-XX参数选项

* 非标准化参数,属于实验性,不稳定,以`-XX`开头;
* -XX参数选项分为两种:
  * Boolean格式:`-XX:+<option>`:表示启用option属性,`-XX:-<option>`:表示禁用option属性;
  * 非Boolean格式:`-XX:<option>=<number>`或者`-XX:<option>=<string>`;

  # 2.添加JVM参数的方法

* 运行Jar包:`java -Xms50m -Xmx50 -jar demo.jar`;
* 通过Tomcat运行war包,在`tomcat/bin/catalina.sh`中添加类似如下配置:`JAVA_OPTS="-Xms512M -Xmx1024M"`;
* 程序运行中:使用`jinfo -flag <name>=<value> <pid>`,`jinfo -flag [+|-]<name> <pid>`,但是可设置的内容有限;

# 3.常用的JVM参数

## 3.1.打印设置的XX选项以及值

* `-XX:+PrintCommandLineFlags`:可以让程序运行前打印出用户手动设置或者JVM自动设置的XX选项;
* `-XX:+PrintFlagsInitial`:表示打印出所有XX选项的默认值;
* `-XX:+PrintFlagsFinal`:表示打印出XX选项在运行程序时生效的值;
* `-XX:+PrintVMOptions`:表示JVM的参数;

## 3.2.内存的选项以及值

### 3.2.1.栈

* `-Xss128k`:设置每个线程的栈大小为128k;

### 3.2.2.堆

* `-Xms3550m`:等价于`-XX:InitialHeapSize`,设置JVM初始堆内存为3550M;
* `-Xmx3550m`:等价于`-XX:MaxHeapSize`,设置JVM最大堆内存3550M;
* `-Xmn2g`:设置年轻代大小2G,官方推荐配置为整个堆大小的3/8;
* `-XX:NewSize=1024m`:设置年轻代初始值为1024M;
* `-XX:MaxNewSize=1024m`:设置年轻代最大值为1024M;
* `-XX:SurvivorRatio=8`:设置年轻代中Eden区与一个Survivor区的比值,默认为8;
* `-XX:+UseAdaptiveSizePolicy`:自动选择各区大小比例;
* `-XX:NewRatio=4`:设置老年代与年轻代的比值;
* `-XX:PretenureSizeThreadshold=1024`:设置让大于此阈值的对象直接分配在老年代,单位为字节,只对Serial,ParNew收集器有效;
* `-XX:MaxTenuringThreshold=15`:新生带每次GC后,还存活的对象年龄+1.当对象的年龄大于设置的这个值时就进入老年代,默认15;
* `-XX:+PrintTenuringDistribution`:让JVM每次MinorGC后打印出当前使用的Survivor中对象的年龄分布;
* `-XX:TargetSurivivor`:表示MinorGC结束后Survivor区域中占用空间的期望比例;

### 3.2.3.方法区

#### 3.2.3.1.永久代

* `-XX:PermSize=256m`:设置永久代初始值为256M;
* `-XX:MaxPermSize=256m`:设置永久代最大值为256M;

#### 3.2.3.2.元空间

* `-XX:MetaSpaceSize`:初始空间大小;
* `-XX:MaxMetaspaceSize`:最大空间,默认没有限制;
* `-XX:+UseCompressedOops`:压缩对象指针;
* `-XX:+UseCompressedClassPointers`:压缩类指针;
* `-XX:+CompressedClassSpaceSize`:设置Klass Metaspace的大小,默认1G;

### 3.2.4.直接内存

* `-XX:MaxDirectMemorySize`:指定DirectMemory容量,若未指定,默认与Java堆最大值一样;

## 3.3.OOM相关的选项以及值;

* `-XX:+HeapDumpOnOutOfMemoryError`:表示在内存出现OOM的时候,把Heap Dump到文件;
* `-XX:+HeapDumpBeforeFullGC`:表示在出现FullGC之前,生成Heap转储文件;
* `-XX:HeapDumpPath=<path>`:指定heap转存文件的存储路径;
* `-XX:OnOutOfMemoryError`:指定一个可行性程序或者脚本路径,当发生OOM的时候,去执行;

## 3.4.垃圾收集器的选项以及值

### 3.4.1.查看默认垃圾回收器 

* `-XX:+PrintCommandLineFlags`:查看命令行参数(包含使用的垃圾收集器);
* `jinfo -flag <垃圾收集器参数> <进程ID>`;

### 3.4.2.Serial回收器

* `-XX:+UseSerialGC`:指定年轻代和老年代都使用串行收集器.年轻代用Serial GC,老年代用Serial Old GC;

### 3.4.3.ParNew回收器

* `-XX:+UseParNewGC`:指定使用ParNew收集器执行内存回收任务,表示年轻代使用并行收集器,不影响老年代;
* `-XX:ParallelGCThreads=N`:限制线程数量,默认开启和CPU相同的数量;

### 3.4.4.Parallel回收器

* `-XX:+UseParallelGC`:手动指定年轻代使用Parallel并行收集器;
* `-XX:+UseParallelOldGC`:手动指定老年代使用并行回收收集器;
* 上面两个参数互相激活;
* `-XX:ParallelGCThreads`:设置年轻代并行收集器的线程数,一般最好与CPU数量相等;
* `-XX:MaxGCPauseMillis`:设置垃圾收集器的最大停顿时间,单位是毫秒,设置该参数后,收集器在工作时会调整Java堆大小或其他参数,该参数使用需谨慎;
* `-XX:GCTimeRatio`:垃圾收集时间占总时间的比例`=1(N+1)`;用于衡量吞吐量大小;
  * 取值(0,100),默认99,垃圾回收时间不超过1%;
  * 与`-XX:MaxGCPauseMillis`参数有一定矛盾性;
* `-XX:+UseAdaptiveSizePolicy`:设置Parallel收集器有自适应调节策略;
  * 在手动调优比较困难的场合,可以直接使用自适应的方式,仅指定虚拟机的最大堆,目标吞吐量和暂停时间,让虚拟机自己完成调优工作;

### 3.4.5.CMS回收器

* `-XX:+UseConcMarkSweepGC`:手动指定使用CMS收集器执行内存回收任务;
  * 开启该参数后,会激活`-XX:+UseParNewGC`,即使用:ParNew+CMS+Serial Old组合;
* `-XX:CMSInitiatingOccupanyFraction`:设置堆内存使用的阈值,一旦达到该阈值,便开始进行回收;JDK6以上默认是92%;
  * 如果内存增长缓慢,可以设置一个稍大的值,大的阈值可以有效降低CMS的触发频率,如果内存增长很快,应该降低,避免频繁触发Serial Old,有效降低Full GC的次数;
* `-XX:+UseCMSCompactAtFullCollection`:用于指定在执行完Full GC后对内存空间进行压缩,由于内存压缩无法并发执行,停顿时间更长;
* `-XX:CMSFullGCsBeforeCompaction`:设置在执行多少次GC后对内存进行压缩;
* `-XX:ParallelCMSThreads`:设置CMS的线程数量;

### 3.4.6.G1回收器

* `-XX:+UseG1GC`:手动使用G1收集器;
* `-XX:G1HeapRegionSize`:设置每个Region的大小,值是2的幂,范围是1MB到32MB之间,目标是根据最小的Java堆大小划分出2048个区域,默认是堆内存的1/2000;
* `-XX:MaxGCPauseMillis`:设置期望达到的最大GC停顿时间JVM尽力实现,默认值是200ms;
* `-XX:ParallelGCThread`:设置STW时候GC线程数的值,最多8;
* `-XX:ConcGCThreads`:设置并发标记的线程数,n设置为ParallelGCThreads的1/4左右;
* `-XX:InitiatingHeapOccupancyPercent`:设置触发并发GC周期的Java堆占用率阈值,超过此值,触发GC,默认是45;
* `-XX:G1NewSizePercent`:新生代占用堆内存的最小比例;
* `-XX:G1MaxNewSizePercent`:新生带占用堆内存的最大比例;
* `-XX:G1ReservePercent`:保留内存区域,防止to space溢出;

## 3.5.GC日志相关的选项以及值

* `-verbose:gc`:输出GC日志信息,默认输出到标准输出;
* `-XX:+PrintGC`:等同于`-verbose:gc`;
* `-XX:+PrintGCDetails`:在发生垃圾回收时详细的日志,并在进程退出时候输出当前内存各个区域分配的情况;
* `-XX:+PrintGCTimeStamps`:输出GC发生时的时间戳,不可以独立使用
* `-XX:+PrintGCDateStamps`:输出GC发生时的时间戳.以日期的格式,不可以独立使用;
* `-XX:+PrintHeapAtGC`:每次GC前与GC后,都打印堆信息;
* `-Xloggc:<file>`:把GC日志写入到一个文件中去,而不是打印到标准输出中;

# 4.Java语言获取JVM参数

~~~java
public class MemoryMonitor {
    public static void main(String[] args) {
        MemoryMXBean memorymbean = ManagementFactory.getMemoryMXBean();
        MemoryUsage usage = memorymbean.getHeapMemoryUsage();
        System.out.println("INIT HEAP: " + usage.getInit() / 1024 / 1024 + "m");
        System.out.println("MAX HEAP: " + usage.getMax() / 1024 / 1024 + "m");
        System.out.println("USE HEAP: " + usage.getUsed() / 1024 / 1024 + "m");
        System.out.println("\nFull Information:");
        System.out.println("Heap Memory Usage: " + memorymbean.getHeapMemoryUsage());
        System.out.println("Non-Heap Memory Usage: " + memorymbean.getNonHeapMemoryUsage());

        System.out.println("=======================通过java来获取相关系统状态============================ ");
        System.out.println("当前堆内存大小totalMemory " + (int) Runtime.getRuntime().totalMemory() / 1024 / 1024 + "m");// 当前堆内存大小
        System.out.println("空闲堆内存大小freeMemory " + (int) Runtime.getRuntime().freeMemory() / 1024 / 1024 + "m");// 空闲堆内存大小
        System.out.println("最大可用总堆内存maxMemory " + Runtime.getRuntime().maxMemory() / 1024 / 1024 + "m");// 最大可用总堆内存大小

    }
}
~~~

