# JVM-3.虚拟机性能监控,故障处理工具

# 1.基础故障处理工具

## 1.1.jps:虚拟机进程状况工具

jps(JVM Process Status Tool):可以列出正在运行的虚拟机进程,并显示虚拟机执行主类(Main Class,main()函数所在的类)名称以及这些进程的本地虚拟机唯一ID(LVMID,Local Virtual Machine Identifier).对于本地虚拟机进程来说,LVMID与操作系统的进程ID(PID Process Identifier)是一致的.

![image-20210429155108978](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429155108978.png)

| 选项 | 作用                                                |
| ---- | --------------------------------------------------- |
| -q   | 只输出LVMID,省略主类的名称;                         |
| -m   | 输出虚拟机进程启动时传递给主类`main()`函数的参数    |
| -l   | 输出主类的全名,如果进程执行的是JAR包,则输出JAR路径; |
| -v   | 输出虚拟机进程启动时的JVM参数                       |

## 1.2.jstat:虚拟机统计信息监视工具

jstat(JVM statistics Monitoring Tool)是用于监视虚拟机各种运行状态信息的命令行工具,他可以显示本地或者远程虚拟机进程中类加载,内存,垃圾收集,即使编译等运行数据.

本地的虚拟机进程:`jstat [ generalOption | outputOptions vmid [ interval[s|ms] [ count ] ]`,如果是远程虚拟机进程,VMID的格式应该是:`[protocol:][//]lvmid[@hostname[:port]/servername]`

选项option代表用户希望查询的虚拟机信息,主要分为三类,类加载,垃圾收集,运行期编译状况:

 ![image-20210429163748053](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429163748053.png)

![image-20210429163957808](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429163957808.png)

E表示Eden区,S0和S1表示两个Survivor区.O表示老年代,M表示元数据空间,YGC表示Young GC的次数,YGCT表示Young GC的时间.FGC表示Full GC的次数,FGCT表示Full GC的时间,GCT表示GC的总耗时;

## 1.3.jinfo:java配置信息

jinfo(Configuration Info for Java)的作用是实时查看和调整虚拟机各项参数.如果想知道未被显式指定的参数的系统默认值,可以使用`jinfo -flags <pid>`来进行查询,如果JDK6及其以上.使用`java -XX:+PrintFlagsFinal`来查看参数默认值.

![image-20210429165805716](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429165805716.png)

## 1.4.jmap:java内存映射工具

jmap(Memory Map for Java)命令用于生成堆转存储快照(一般称为heapdump或dump文件),还可以查询finalize执行队列,Java堆和方法区的详细信息.如空间使用率,当前用的是哪种收集器.jmap命令格式:`jmap [option] vmid`,option的值如下:

![image-20210429171357544](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429171357544.png)

![image-20210429171628695](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429171628695.png)

## 1.5.jhat:虚拟机堆转储快照分析工具

JDK提供jhat(JVM Heap Analysis Tool)命令与jmap搭配使用,来分析jmap生成的堆存储快照.

![image-20210429172501089](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429172501089.png)

![image-20210429172644044](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429172644044.png)

jhat的分析功能相对比较简陋,大部分使用VisualVM,Eclipse Mempory Analyzer,IBM HeapAnalyzer;

![image-20210429173159251](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429173159251.png)

分析结果主要以包为单位进行分组显示,分析内存泄漏问题主要会用到其中的"Heap Histogram"与QQL页签的功能.

## 1.6.jstack:Java堆栈跟踪工具

jstack(Stack Trace for Java)命令用于生成虚拟机当前时刻的线程快照(一般称为threaddump或者javacore文件).线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合,生成线程快照的目的通常是定位线程出现长时间停顿的原因,如线程间死锁,死循环,请求外部资源导致的长时间挂起,都是导致线程长时间停顿的常见原因.

`jstack [option] vmid`;

option选项的和合法值如下:

| 选项 | 作用                                           |
| ---- | ---------------------------------------------- |
| -F   | 当正常输出的请求不被响应时候,强制输出线程堆栈. |
| -l   | 除堆栈外,显示关于锁的附加信息                  |
| -m   | 如果调用到本地方法,可以显示C/C++的堆栈         |

## 1.7.基础工具列表

参考<深入理解java虚拟机 第三版>148-151页;

# 2. 可视化故障处理工具

## 2.1.JHSDB:基于服务性代理的调试工具

1.8无法使用???

![image-20210429180038911](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429180038911.png)

## 2.2.JConsole:Java监视与管理控制台

Jconsole(Java Monitoring and Management Console)是一款基于JMX(Java Management Extensions)的可视化监视,管理工具.它的主要功能是通过JMX的MBean(ManagedBean)对系统进行信息收集和参数动态调整.JMX是一种开放性技术,不仅可以用在虚拟机本身的管理,还可以运行于虚拟机之上的软件中.典型的如中间件大多也基于JMX来实现管理与监控.虚拟机对JMX MBean的访问也是完全开放的,可以使用代码调用API,支持JMX协议的管理控制台,或者其他符合JMX规范的软件进行访问.

### 2.2.1.内存监控

相当于可视化的jstat命令

~~~java
/**
 * @Author:guoqing.zhu
 * @Date：2021/4/29 19:21
 * @Desription: -Xms100m -Xmx100m -XX:+UseSerialGC
 * @Version: 1.0
 */
public class OOMObject {
    static class OOMObj{
        public byte[] placeholder = new byte[64*1024];
    }

    public static void fillHeap(int num) throws InterruptedException {
        ArrayList<OOMObj> oomObjs = new ArrayList<>();
        for (int i = 0; i < num; i++) {
            Thread.sleep(50);
            oomObjs.add(new OOMObj());
        }
        System.gc();
    }

    public static void main(String[] args) throws InterruptedException {
        fillHeap(1000);
    }
}
~~~

![image-20210429194133676](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429194133676.png)

![image-20210429194159493](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429194159493.png)

### 2.2.2.线程监控

相当于可视化jstack命令.线程长时间停顿的主要原因有等待外部资源(数据库连接,网络资源,设备资源),死循环,锁等待等;

```java
/**
 * @Author:guoqing.zhu
 * @Date：2021/4/29 19:46
 * @Desription: TODO
 * @Version: 1.0
 */
public class ThreadJconsole {
    /**
     * 线程死循环
     */
    public static void createBusyThread(){
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while(true){

                }
            }
        },"testBusyThread");
        thread.start();
    }

    public static void createLockThread(final Object lock){
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (lock){
                    try{
                        lock.wait();
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
            }
        },"testLockThread");
        thread.start();
    }

    public static void main(String[] args) {
        createBusyThread();
        //createLockThread(new ThreadJconsole());
    }
}
```

执行`createBusyThread()`时如下:从堆栈追踪中看到一直在代码第15行停留,就是`ThreadJconsole`的第15行就是`while()`循环中停留,所以会在空循环耗尽操作系统分配给它的执行时间,直到线程切换为止.

![image-20210429195705082](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429195705082.png)

执行`createLockThread()`如下:线程这时候处于WAITING状态;

![image-20210429200150985](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429200150985.png)

死锁检测

~~~java
public class DeadLockJconsole {

    static class SynAddRunnable implements Runnable{

        int a,b;

        public SynAddRunnable(int a,int b){
            this.a = a;
            this.b = b;
        }

        @Override
        public void run() {
            synchronized (Integer.valueOf(a)){
                synchronized (Integer.valueOf(b)){
                    System.out.println(a+b);
                }
            }
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new Thread(new SynAddRunnable(1,2)).start();
            new Thread(new SynAddRunnable(2,1)).start();
        }
    }
}
~~~

![image-20210429201128336](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429201128336.png)

出现死锁之后,将会出现死锁页签;

## 2.3.VisualVM:多合-故障处理器

VisualVM是基于NetBeans平台开发工具,所以一开始它具备了通过插件功能的能力.有了插件支持.可以做到:

1. 显示虚拟机进程以及进程的配置,环境信息(jps,jinfo);
2. 监视应用层需的处理器,垃圾收集,堆,方法区以及线程信心(jstat,jstack);
3. dump以及分析堆转储快照(jmap,jhat);
4. 方法级的程序运行性能分析,找出被调用最多,运行时间最长的方法;
5. 你先程序快照,收集程序的运行时配置,线程dump,内存dump等信息建立一个快照,可以将快照发送开发者进行Bug反馈;
6. 其他插件带来无限可能;

控制台`jvisualvm`打开,可以在[插件中心](https://visualvm.github.io/pluginscenters.html)下载插件,也可以如下图所示安装插件,也可以将插件下载到本地,选择'已下载'的本地插件安装.

![image-20210430094152011](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210430094152011.png)

### 2.3.1.生成,浏览堆转储快照

![image-20210430110856728](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210430110856728.png)

选择应用程序,右键单机应用程序,选择"堆Dump"

![image-20210430111027695](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210430111027695.png)

选择另存为,即可保存快照到本地,通过文件菜单的装入,选择硬盘上的文件就可以浏览.

### 2.3.2.分析程序性能

在Profiler页签中,VisualVM提供了程序运行期间方法级的处理器执行时间分析以及内存分析.做Profiling分析肯定会对程序运行性能有较大影响,所以一般不在生产环境使用该功能,或者改为JMC来完成.

### 2.3.3.BTrace动态日志跟踪

未能理解

## 2.4.Java Mission Control

控制台输入`jmc`打开,发现无法打开,直接去官网下载最新版本的https://www.oracle.com/java/technologies/jdk-mission-control.html

mac无法打开,具体内容查看<深入理解Java虚拟机 第三版>171页;

