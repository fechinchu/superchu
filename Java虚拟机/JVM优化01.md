# JVM优化01

# 1.为什么对JVM做优化

在本地开发环境中我们很少会需要对JVM进行优化的需求,但是到了生产环境,我们可能将有下面的需求:

* 运行的应用"卡住了",日志不输出,程序没有反应;
* 服务器的CPU负载突然升高;
* 在多线程应用下.如何分配线程的数量;

# 2.JVM的运行参数

## 2.1.三种参数类型

JVM的参数类型分为三类,分别是:

* 标准参数:
  * `-help`
  * `-version`
* -X参数(非标准参数)
  * `-Xint`
  * `-Xcomp`
* -XX参数(使用率较高)
  * `-XX:newSize`
  * `-XX:+UseSerialGC`

## 2.2.标准参数

JVM的标准参数,一般都是很稳定的,在未来的JVM版本中不会改变,可以使用`java -help`检索出所有标准参数;

`java -help`
用法: `java [-options] class [args...]`
           (执行类)
   或 ` java [-options] -jar jarfile [args...]`
           (执行 jar 文件)
其中选项包括:
    `-d32`	  使用 32 位数据模型 (如果可用)
    `-d64`	  使用 64 位数据模型 (如果可用)
   ` -server`	  选择 "server" VM
                  默认 VM 是 server,
                  因为您是在服务器类计算机上运行。


```shell
-cp <目录和 zip/jar 文件的类搜索路径>
-classpath <目录和 zip/jar 文件的类搜索路径>
              用 : 分隔的目录, JAR 档案
              和 ZIP 档案列表, 用于搜索类文件。
-D<名称>=<值>
              设置系统属性
-verbose:[class|gc|jni]
              启用详细输出
-version      输出产品版本并退出
-version:<值>
              警告: 此功能已过时, 将在
              未来发行版中删除。
              需要指定的版本才能运行
-showversion  输出产品版本并继续
-jre-restrict-search | -no-jre-restrict-search
              警告: 此功能已过时, 将在
              未来发行版中删除。
              在版本搜索中包括/排除用户专用 JRE
-? -help      输出此帮助消息
-X            输出非标准选项的帮助
-ea[:<packagename>...|:<classname>]
-enableassertions[:<packagename>...|:<classname>]
              按指定的粒度启用断言
-da[:<packagename>...|:<classname>]
-disableassertions[:<packagename>...|:<classname>]
              禁用具有指定粒度的断言
-esa | -enablesystemassertions
              启用系统断言
-dsa | -disablesystemassertions
              禁用系统断言
-agentlib:<libname>[=<选项>]
              加载本机代理库 <libname>, 例如 -agentlib:hprof
              另请参阅 -agentlib:jdwp=help 和 -agentlib:hprof=help
-agentpath:<pathname>[=<选项>]
              按完整路径名加载本机代理库
-javaagent:<jarpath>[=<选项>]
              加载 Java 编程语言代理, 请参阅 java.lang.instrument
-splash:<imagepath>
              使用指定的图像显示启动屏幕
```
### 2.2.1.通过-D设置系统属性参数

~~~java
public class TestJvm {
    public static void main(String[] args) {
        String str = System.getProperty("str");
        if(str == null){
            System.out.println("nulllll");
        }else {
            System.out.println(str);
        }
    }
}
~~~

![image-20200523153926679](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523153926679.png)

### 2.2.2.-server与-client参数

可以通过-server或-client设置jvm的运行参数;

* 它们的区别是Server VM的初始堆空间会大一些,默认使用的是并行垃圾回收器,启动慢运行块;
* Client VM相对来说会保守一些,初始堆空间会小一些,使用串行的垃圾回收器,它的目标是为了让JVM的启动速度更快,但运行速度会比Server模式慢;
* JVM在启动的时候会根据硬件和操作系统自动选择使用Server还是Client类型的JVM;
  * 32位操作系统
    * 如果是Windows操作系统,不论硬件配置如何,都默认使用Client类型的JVM;
    * 如果是其他操作系统上,机器配置有2GB以上的内存同时有2个以上CPU的话默认使用Server模式,否则使用client模式;
  * 64为操作系统
    * 只有Server类型,不支持client类型;

![image-20200523155038820](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523155038820.png)

## 2.3.-X参数

~~~shell
➜  ~ java -X
    -Xmixed           混合模式执行 (默认)
    -Xint             仅解释模式执行
    -Xbootclasspath:<用 : 分隔的目录和 zip/jar 文件>
                      设置搜索路径以引导类和资源
    -Xbootclasspath/a:<用 : 分隔的目录和 zip/jar 文件>
                      附加在引导类路径末尾
    -Xbootclasspath/p:<用 : 分隔的目录和 zip/jar 文件>
                      置于引导类路径之前
    -Xdiag            显示附加诊断消息
    -Xnoclassgc       禁用类垃圾收集
    -Xincgc           启用增量垃圾收集
    -Xloggc:<file>    将 GC 状态记录在文件中 (带时间戳)
    -Xbatch           禁用后台编译
    -Xms<size>        设置初始 Java 堆大小
    -Xmx<size>        设置最大 Java 堆大小
    -Xss<size>        设置 Java 线程堆栈大小
    -Xprof            输出 cpu 配置文件数据
    -Xfuture          启用最严格的检查, 预期将来的默认值
    -Xrs              减少 Java/VM 对操作系统信号的使用 (请参阅文档)
    -Xcheck:jni       对 JNI 函数执行其他检查
    -Xshare:off       不尝试使用共享类数据
    -Xshare:auto      在可能的情况下使用共享类数据 (默认)
    -Xshare:on        要求使用共享类数据, 否则将失败。
    -XshowSettings    显示所有设置并继续
    -XshowSettings:all
                      显示所有设置并继续
    -XshowSettings:vm 显示所有与 vm 相关的设置并继续
    -XshowSettings:properties
                      显示所有属性设置并继续
    -XshowSettings:locale
                      显示所有与区域设置相关的设置并继续

-X 选项是非标准选项, 如有更改, 恕不另行通知。


以下选项为 Mac OS X 特定的选项:
    -XstartOnFirstThread
                      在第一个 (AppKit) 线程上运行 main() 方法
    -Xdock:name=<应用程序名称>"
                      覆盖停靠栏中显示的默认应用程序名称
    -Xdock:icon=<图标文件的路径>
                      覆盖停靠栏中显示的默认图标
~~~

### 2.3.1. -Xint,-Xcomp,-Xmixed

* 在解释模式(interpreted mode)下,-Xint标记会强制JVM执行所有的字节码,当然这会降低运行速度,通常低10被或更多;
* `-Xcomp`参数与`-Xint`正好相反,JVM在第一次使用时会把所有字节码编译成本地代码,从而带来最大程度的优化;
  * 然而,很多应用在`-Xcomp`也会有一些性能损失,当然这比使用`-Xint`损失的少,原因是`-Xcomp`没有让JVM启用JIT编译器的全部功能.JIT编译器可以对是否需要编译做判断,如果所有代码都进行编译的话,对于一些只执行一次的代码就没有意义了;
* `-Xmixed`是混合模式,将解释模式与编译模式进行混合使用,由JVM自己决定,这是JVM默认的模式,也是推荐使用的模式;

## 2.4. -XX参数

`-XX`参数也是非标准参数,主要用于JVM调优和debug操作;

`-XX`参数的使用有2种方式,一种是boolean类型,一种是非boolean类型;

* boolean类型
  * 格式:`-XX:[+-]<name>`表示启动或禁用<name>属性;
    * 如:`-XX:+DisableExplicitGC`表示禁用手动调用gc操作,也就是说调用System.gc()无效;
* 非boolean类型
  * 格式:`-XX:<name>=<value>`表示<name>属性值为<value>;
    * 如:`-XX:NewRatio=1`表示新生代和老年代的比值;

## 2.5. -Xms与-Xmx参数

* `-Xms`与`-Xmx`分别是设置Jvm的堆内存的初始值大小和最大大小;

* `-Xmx2048m`:等价于`-XX:MaxHeapSize`,设置JVM最大堆内存为2048M;

* `-Xms512m`:等价于`-XX:InitialHeapSize`,设置JVM初始堆内存为512M;

## 2.6.查看JVM的运行参数

### 2.6.1.运行Java命令时打印参数

运行Java命令时打印参数,需要添加`-XX:+PrintFlagsFinal`参数即可;

![image-20200523164424615](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523164424615.png)

由上述的信息可以看出,参数有boolean类型和数字类型,值的操作符是`=`和`:=`,分别代表默认值和被修改的值;

### 2.6.2.查看正在运行的JVM参数

如果想要查看正在运行的JVM就需要借助于jinfo命令查看;

首先我们可以使用`jps -l`查看所有的java进程;

![image-20200523170852254](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523170852254.png)

```shell
jinfo -flag <参数名> <进程id>
```

如果我们想要查看某一参数的值,用法如上;

# 3.JVM的内存模型

## 3.1.JDK1.7的堆内存模型

![image-20200523171905462](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523171905462.png)

* Young年轻区

  Young区被划分为三部分,Eden区和两个大小严格相同的Survivor区.其中,Survivor区间中,某一时刻只有其中一个是被使用的,另外一个留做垃圾收集时复制对象用,在Eden区间变满的时候,GC就会将存活的对象移到空闲的Survivor区间中,根据JVM的策略,在经过几次垃圾收集后,仍然存活于Survivor的对象将被移动到Tenured区间;

* Tenured年老区

  Tenured区主要保存生命周期长的对象,一般是一些老的对象,当一些对象在Young复制转移一定的次数以后,对象就会被转移到Tenured区,一般如果系统中用了application级别的缓存,缓存中的对象往往会被转移到这一区间;

* Perm永久区

  Perm代主要保存class,method,filed对象,这部分的空间一般不会溢出,除非一次性加载加载了很多的类,不过在涉及到了部署的应用服务器的时候,有时候会遇到`java.lang.OutOfMemoryError : PermGen space`的错误,造成这个错误的很大原因就有可能是每次都重新部署,但是重新部署之后,类的class没有被卸载掉,这样就造成了大量的class对象保存在了perm中,这种情况下,一般重新启动应用服务器可以解决问题;

* Virtual区

  最大内存和初始内存的差值就是VIrtual区;

## 3.2.JDK1.8的堆内存模型

![image-20200523173336941](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523173336941.png)

由上图可以看出,jdk1.8的内存模型是由2部分组成,年轻代+年老代

年轻代:Eden+2*Survivor

年老代:OldGen

在JDK1.8中变化最大的是Perm区,用Metaspace(元数据空间)进行了替换;

需要特别说明的是:Metaspace所占用的内存空间不是在虚拟机内部,而是在本地内存空间中,这也是与1.7的永久代最大的区别所在;

![image-20200523173748767](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523173748767.png)

## 3.3.为什么要废弃1.7中的永久区

http://openjdk.java.net/jeps/122

> This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.
>
> 移除永久代是为融合HotSpot JVM与 JRockit VM而做出的努力，因为JRockit没有永久代，不需要配置永久代。

现实使用中,由于永久代内存经常不够用或发生内存泄露,爆出异常`java.lang.OutOfMemoryError: PermGen`.基于此,将永久区废弃,而改用元空间,改为了使用本地内存空间.

## 3.4.通过jstat命令进行查看堆内存使用情况

`jstat`可以查看堆内存各部分的使用量,以及加载类的数量.命令的格式如下:

`jstat [-命令选项] [vmid] [间隔时间/毫秒] [查询次数]`;

### 3.4.1.查看class加载统计

~~~shell
[root@fechinchu ~]# jstat -class 1970
Loaded  Bytes  Unloaded  Bytes     Time   
  8419 16681.0       64   140.9      12.12
~~~

说明:

* Loaded:加载class的数量;
* Bytes:所占用空间大小;
* Unloaded:未加载数量;
* Bytes:未加载占用空间;
* Time:时间;

### 3.4.2.查看编译统计

~~~shell
[root@fechinchu ~]# jstat -compiler 1970
Compiled Failed Invalid   Time   FailedType FailedMethod
    5794      3       0    12.63          1 com/google/inject/internal/cglib/core/$MethodWrapper$MethodWrapperKey$$$KeyFactoryByCGLIB$$e23ecb5d hashCode
~~~

说明:

* Compiled:编译数量;
* Failed:失败数量;
* Invalid:不可用数量;
* FailedType:失败类型;
* FailedMethod:失败的方法;

### 3.4.3.垃圾回收统计

~~~shell
[root@fechinchu ~]# jstat -gc 1970
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
1024.0 3584.0 640.0   0.0   462336.0 117410.8  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
  
#也可以指定打印的时间和次数,每1s打印一次,共打印5次
[root@fechinchu ~]# jstat -gc 1970 1000 5
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
1024.0 3584.0 640.0   0.0   462336.0 141790.5  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
1024.0 3584.0 640.0   0.0   462336.0 141790.5  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
1024.0 3584.0 640.0   0.0   462336.0 141790.5  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
1024.0 3584.0 640.0   0.0   462336.0 141790.5  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
1024.0 3584.0 640.0   0.0   462336.0 141790.5  68608.0    42911.8   50776.0 48703.4 6272.0 5686.6     56    1.026   3      0.254    1.280
~~~

* S0C:第一个Survivor区的大小(KB);
* S1C:第二个Survivor区的大小(KB);
* S0C:第一个Survivor区的使用大小(KB);
* S1C:第二个Survivor去的使用大小(KB);
* EC:Eden区的大小(KB);
* EU:Eden区的使用大小(KB);
* OC:Old区大小(KB);
* OU:Old区使用大小(KB);
* MC:方法区大小(KB);
* MU:方法区使用大小(KB);
* CCSC:压缩类空间大小(KB);
* CCSU:压缩类空间使用大小(KB);
* YGC:年轻代垃圾回收次数;
* YGCT:年轻代垃圾回收消耗时间;
* FGC:老年代垃圾回收次数;
* FGCT:老年代垃圾回收消耗时间;
* GCT:垃圾回收消耗总时间;

# 4.jmap的使用以及内存溢出分析

## 4.1.查看内存使用情况

~~~shell
[root@fechinchu ~]# jmap -heap 1970
Attaching to process ID 1970, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.171-b11

using thread-local object allocation.
Parallel GC with 2 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 1518338048 (1448.0MB)
   NewSize                  = 31981568 (30.5MB)
   MaxNewSize               = 505937920 (482.5MB)
   OldSize                  = 64487424 (61.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 473432064 (451.5MB)
   used     = 163087560 (155.53241729736328MB)
   free     = 310344504 (295.9675827026367MB)
   34.44793295622664% used
From Space:
   capacity = 1048576 (1.0MB)
   used     = 655360 (0.625MB)
   free     = 393216 (0.375MB)
   62.5% used
To Space:
   capacity = 3670016 (3.5MB)
   used     = 0 (0.0MB)
   free     = 3670016 (3.5MB)
   0.0% used
PS Old Generation
   capacity = 70254592 (67.0MB)
   used     = 43941672 (41.906044006347656MB)
   free     = 26312920 (25.093955993652344MB)
   62.54633433783232% used

22938 interned Strings occupying 1898992 bytes.
~~~

## 4.2.查看内存中对象数量及大小

![image-20200523182715746](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523182715746.png)

~~~shell
#查看所有对象,包括活跃以及非活跃的
jmap -histo <pid> | more

#查看活跃对象
jmap -histo:live <pid> | more
~~~

> 对象说明:
>
> B:byte
>
> C:char
>
> D:double
>
> F:float
>
> I:int
>
> J:long
>
> Z:boolean
>
> [:数组
>
> [L+:类名 其他对象

## 4.3.将内存使用情况dump到文件中

有些时候我们需要将jvm当前内存中的情况dump到文件中,然后对它进行分析,jmap也是支持dump到文件中的.

~~~shell
#用法
jmap -dump:format=b,file=<dumpFileName> <pid>
#示例
jmap -dump:format=b,file=/tmp/dump.dat 1970
~~~

![image-20200523184233266](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523184233266.png)

## 4.4.通过jhat对dump文件进行分析

![image-20200523184338589](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523184338589.png)

~~~shell
#用法
jhat -port <port> <file>
~~~

之后我们就可以使用浏览器进行访问了

![image-20200523184502697](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523184502697.png)

在最页面的最下面有OQL功能

![image-20200523184617122](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523184617122.png)

## 4.5.通过MAT对dump文件进行分析

~~~shell
➜  MacOS ./MemoryAnalyzer -data dump ~/Develop/dump.dat
~~~



# 5.jstack的使用

有些时候我们需要查看下jvm的线程执行情况,比如,发现服务器的CPU的负载突然增高了,出现了死锁,死循环等,我们该如何分析呢?

由于程序是正常运行的,没有任何的输出,从日志方面也看不出任何问题,所以就需要看jvm的内部线程执行情况,然后再进行分析查找出原因 了;

jstack的作用就是将正在运行的jvm的线程情况进行快照,并且打印出来;

~~~shell
jstack <pid>
~~~

## 5.1.线程的状态

![image-20200523210059563](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523210059563.png)

# 6.VisualVM工具的使用

VisualVM,能够监控线程,内存情况,查看方法的CPU时间和内存中对象,已被GC的对象,反向查看分配的堆栈(比100个String对象分别由哪几个对象分配出来的);

VisualVM使用简单,几乎0配置,功能还是比较丰富的,几乎囊括了其他JDK自带命令的所有功能;

* 内存信息;
* 线程信息;
* Dump堆(本地进程);
* Dump线程(本地进程);
* 打开堆Dump.堆Dump可以用jmap来生成;
* 打开线程Dump;
* 生成应用快照(包含内存信息,线程信息等等);
* 性能分析,CPU分析(各个方法调用时间,检查哪些方法耗时多),内存分析(各类对象占用的内存,检查哪些类占用内存多);

## 6.1.启动

![image-20200523213543202](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200523213543202.png)

## 6.2.本地进程(略)

## 6.3.远程进程

VisualJVM不仅是可以监控本地JVM进程,还可以监控远程的JVM进程,需要借助于JMX技术实现;

### 6.3.1.JMX介绍

JMX(Java Management Extensions,即Java管理拓展)是一个为应用程序,设备,系统等植入管理功能的框架.JMX可以跨越一系列异构操作系统平台,系统体系结构和网络传输协议.灵活的开发无缝的系统,网络和服务管理应用;

### 6.3.2.监控远程的tomcat

~~~shell
#在tomcat的bin目录下，修改catalina.sh，添加如下的参数

JAVA_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 - Dcom.sun.management.jmxremote.authenticate=false -
Dcom.sun.management.jmxremote.ssl=false"

#这几个参数的意思是：
#-Dcom.sun.management.jmxremote ：允许使用JMX远程管理 
#-Dcom.sun.management.jmxremote.port=9999 ：JMX远程连接端口
#-Dcom.sun.management.jmxremote.authenticate=false ：不进行身份认证，任何用户都可以连接 #-Dcom.sun.management.jmxremote.ssl=false ：不使用ssl
~~~

这个时候我们可以使用VisualJVM连接远程tomcat;

