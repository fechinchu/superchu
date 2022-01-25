# JVM详解10-JVM监控及诊断工具-命令行篇

# 1.jps-查看正在运行的Java进程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211101184539168.png" alt="image-20211101184539168" style="zoom: 67%;" />

* **显示指定系统内所有的HotSpot虚拟机进程(查看虚拟机进程信息),可用于查询正在运行的虚拟机进程;**
* **对于本地虚拟机进程来说,进程的本地虚拟机ID与操作下同的进程ID是一致的,是唯一的;**

*  `jps [-q] [-mlvV] [<hostid>]`:这些参数无需多述,尝试一下即可.需要说明的是`-mlvV`可以组合使用,但是`-q`不能与`-mlvV`组合使用;

# 2.jstat-查看JVM的统计信息

![image-20211103091436414](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103091436414.png)

* **jstat用于监视虚拟机各种运行状态信息的命令行工具,可以显示本地或者远程虚拟机进程中的类装载,内存,垃圾收集.JIT编译等运行数据;**

![image-20211103092534701](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103092534701.png)

* 通过`man jstat`命令可以清晰的了解命令的结构;
* 通过`jstat`来查看GC比较常用的用法如下:
  * **如果GCT占用Timestamp的时间段的差值过高,那么说明堆中几乎没有可用空间,随时可以抛出OOM异常;**
  * **通过jstat来判断是否存在内存泄漏;**
    * **首先在长时间运行的Java程序中,可以运行`jstat`命令连续获取多行性能数据,并取这几行数据中OU列(已经占用的老年代内存)的最小值;**
    * **然后,每隔一段较长时间重复上一次操作,来获取多组OU最小值,如果这些值上涨,说明该Java程序的老年代内存在不断上涨,意味着无法回收的对象在不断增加,因此很有可能存在内存泄漏;**

![image-20211103111317168](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103111317168.png)

* 关于`S0C,S1C,S0U`等head的意思`man`中已经有详细注释:

![image-20211103104611911](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103104611911.png)

# 3.jinfo-实时查看和修改JVM的配置参数

![image-20211103204218335](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103204218335.png)

`jinfo`不仅可以查看运行时某一个Java虚拟机的实际取值,甚至可以在运行时修改部分参数,并让其生效.但是,并非所有参数都支持动态修改.参数只有被标记为`manageable`的flag可以被实时修改,这个修改能力是极其有限的;

![image-20211103205048972](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103205048972.png)

* `java -XX:+PrintFlagsInitial`:查看所有JVM参数启动的初始值;
* ` java -XX:+PrintFlagsFinal`:查看所有JVM参数的最终值;
* `java -XX:+PrintCommandLineFlags`:查看那些已经被用户或者JVM设置过的详细的XX参数的名称和值;

# 4.jmap-导出内存映像文件和内存使用情况

![image-20211103210421672](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103210421672.png)

* 导出内存映像文件:
  * jmap常见用法:`jmap -dump:live,format=b,file=dump02.hprof 46469`;
  * **当快要OOM的时候自动dump:`-XX:+HeapDumpOnOutOfMemoryError`,`-XX:HeapDumpPath=<filename.hprof>`;**

# 5.jhat-堆分析工具

JDK提供的jhat命令与jmap命令搭配使用,用于分析jmap生成的heap dump文件.jhat内置了一个微型的服务器,生成dump文件的分析结果后,用户可以在浏览器中查看分析结果;

![image-20211103214658998](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103214658998.png)

在浏览器中打开后使用;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103215245870.png" alt="image-20211103215245870" style="zoom:50%;" />

![image-20211103215306263](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103215306263.png)

# 6.jstack-打印JVM中线程快照

jstack用于生成虚拟机指定进程当前时刻的线程快照,线程快照就是当前虚拟机指定进程的每一条线程正在执行的方法堆栈集合;生成线程快照的作用:可用于定位线程出现长时间停顿的原因,如线程间死锁,死循环,请求外部直言导致的长时间等待等;

在thread dump中,需要注意:

* 死锁.Deadlock;
* 等待资源,Waiting on condition;
* 等待获取监视器,Waiting on monitor entry;
* 阻塞,Blocked;

![image-20211103221138811](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103221138811.png)

# 7.jcmd:多功能命令行

在JDK7之后,新增了jcmd,可以用来实现前面除了jstat之外所有命令的功能.比如:导出堆,内存使用,查看Java进程,导出线程信息,执行GC,JVM运行时间等;

![image-20211103223220133](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211103223220133.png)

* 常见用法
  * `jcmd`:列出所有的JVM进程,类似jps;
  * `jcmd pid help`:针对指定进程,列出所有支持的命令;
  * `jcmd pid <具体命令>`:显示指定进程的指令命令;