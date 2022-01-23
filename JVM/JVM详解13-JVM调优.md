# JVM详解13-JVM调优

# 1.OOM

如下是OOM产生的原因及其频率;

![image-20211112164721882](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112164721882.png)

![image-20211112164744763](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112164744763.png)

## 1.1.Java heap space

```java
@RequestMapping("/add")
    public void addObject(){
        System.err.println("add"+peopleSevice);
        ArrayList<People> people = new ArrayList<>();
        while (true){
            people.add(new People());
        }
    }
```

如上代码模拟线上OOM的情况

```shell
java -XX:+PrintGCDetails -XX:MetaspaceSize=64m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdump.hprof -XX:+PrintGCDateStamps -Xms
200M -Xmx200M -Xloggc:log/gc-oomHeap.log -jar demo-0.0.1-SNAPSHOT.jar 
```

发出请求,得如下响应:

![image-20211111222356783](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111222356783.png)

这时候不妨查看日志,通过日志可以看出问题点

### 1.1.1.jvisualvm分析

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111225227794.png" alt="image-20211111225227794" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111225339848.png" alt="image-20211111225339848" style="zoom:50%;" />

### 1.1.2.MAT分析

#### 1.1.2.1.Leak Suspect

在Leak Suspect中可以看到问题点

![image-20211112143037316](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112143037316.png)

#### 1.1.2.2.Thread Overview

![image-20211112145220650](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112145220650.png)

根据Leak Suspect获取到有问题的线程,在Thread Overview中可以看出问题点;

#### 1.1.2.3.Histogram

通过Histogram图中的incoming references也可以看出问题点

![image-20211112143526642](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112143526642.png)

### 1.1.3.GCEasy

https://gceasy.io/来分析GClog;

### 1.1.4.原因以及解决

* 原因
  * 代码中可能存在大对象分配;
  * 可能存在内存泄漏,导致在多次GC之后,还是无法找到一块足够大的内存容纳当前对象;
* 解决方法:
  * 检查是否存在大对象的分配，最有可能的是大数组分配;
  * 通过MAT工具分析是否存在内存泄漏的问题;
  * 如果没有找到明显的内存泄漏，使用 -Xmx 加大堆内存;
  * 检查是否有大量的自定义的 Finalizable 对象，也有可能是框架内部提供的，考虑其存在的必要性

## 1.2.Metaspace

Java虚拟机规范对方法区的限制非常宽松,除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外,还可以选择不实现垃圾收集.垃圾收集行为在这个区域是比较少出现的,**其内存回收目标主要是针对常量池的回收和对类型的卸载**.当方法区无法满足内存分配需求时,将抛出OutOfMemoryError;

```java
@RequestMapping("/metaSpaceOom")
    public void metaSpaceOom(){
        ClassLoadingMXBean classLoadingMXBean = ManagementFactory.getClassLoadingMXBean();
        while (true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(People.class);
            enhancer.setUseCache(false);
            //enhancer.setUseCache(true);
            enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) -> {
                System.out.println("我是加强类，输出print之前的加强方法");
                return methodProxy.invokeSuper(o,objects);
            });
            People people = (People)enhancer.create();
            people.print();
            System.out.println(people.getClass());
            System.out.println("totalClass:" + classLoadingMXBean.getTotalLoadedClassCount());
            System.out.println("activeClass:" + classLoadingMXBean.getLoadedClassCount());
            System.out.println("unloadedClass:" + classLoadingMXBean.getUnloadedClassCount());
        }
    }
```

```shell
java -XX:+PrintGCDetails -XX:MetaspaceSize=60m -XX:MaxMetaspaceSize=60m -Xss512K -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdumpMeta.hprof -XX:SurvivorRatio=8 -XX:+TraceClassLoading -XX:+TraceClassUnloading -XX:+PrintGCDateStamps -Xms60M -Xmx60M -Xloggc:log/gc-oomMeta.log demo-0.0.1-SNAPSHOT.jar
```

### 1.2.1.jvisualvm分析

![image-20211112165931772](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112165931772.png)

### 1.2.2.MAT分析

在Histogram中group by package;可以看到用了大量的反射,创建了太多的类.导致Metaspace OOM;

![image-20211115094439530](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115094439530.png)

### 1.2.3.GCEasy

![image-20211115095413077](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115095413077.png)

### 1.2.4.原因以及解决

* 原因:
  * 运行期间生成了大量的代理类,导致方法区被撑爆,无法卸载;
  * 应用长时间运行,没有重启;
  * 元空间内存设置过小;
* 解决方法:
  * 检查是否永久代或者元空间设置的过小;
  * 检查代码中是否存在大量的反射操作;
  * dump之后通过MAT检查是否存在大量由于反射生成的代理类;

## 1.3.GC overhead limit exceeded

```java
public static void test1() {
        int i = 0;
        List<String> list = new ArrayList<>();
        try {
            while (true) {
                list.add(UUID.randomUUID().toString().intern());
                i++;
            }
        } catch (Throwable e) {
            System.out.println("************i: " + i);
            e.printStackTrace();
            throw e;
        }
    }
```

### 1.3.1.jvisulavm

![image-20211115104419274](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115104419274.png)

### 1.3.2.MAT分析

![image-20211115110448758](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115110448758.png)

### 1.3.3.原因以及解决

* 原因:
  * 这个是JDK6新加的错误类型,一般是堆太小导致的.Sun官方对此的定义:超过98%的时间用来做GC并且回收了不到2%的堆内存会抛出此异常.本质是一个预判性的异常,抛出该异常时系统没有真正的内存溢出;
* 解决方法:
  * 检查项目中是否有大量的死循环或有使用大内存的代码，优化代码;
  * dump内存，检查是否存在内存泄漏，如果没有，加大内存;

## 1.4.unable to create new native Thread

* 出现这种异常,基本上都是创建了大量的线程导致的;

### 1.4.1.分析

* 通过`-Xss`设置每个线程栈大小的容量;
* JDK5.0以后每个线程堆栈大小为1M,以前每个线程堆栈大小为256K;
* 正常情况下,在相同物理内存下,减少这个值能够生成更多的线程,但是操作系统对一个进程内的线程数还是有限制的,不能无限生成;
* `(MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize) = Number of threads `
  * MaxProcessMemory 指的是进程可寻址的最大空间;
  * JVMMemory JVM内存;
  * ReservedOsMemory 保留的操作系统内存;
  * ThreadStackSize 线程栈的大小;
* 在Java语言里， 当你创建一个线程的时候，虚拟机会在JVM内存创建一个Thread对象同时创建一个操作系统线程，而这个系统线程的内存用的不是JVMMemory，而是系统中剩下的内存`(MaxProcessMemory - JVMMemory - ReservedOsMemory)`;
* 由公式得出结论：你给JVM内存越多，那么你能创建的线程越少，越容易发生`java.lang.OutOfMemoryError: unable to create new native thread`;
* 线程总数也收到系统空闲内存和操作系统的限制,检查是否该系统下有此限制;
  * `/proc/sys/kernel/pid_max`:系统最大pid值，在大型系统里可适当调大 ;
  * `/proc/sys/kernel/threads-max`:系统允许的最大线程数;
  * `maxuserprocess(ulimit -u)`:系统限制某用户下最多可以运行多少进程或线程了;
  * `/proc/sys/vm/max_map_count`:max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。 调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错 ，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用;

![image-20211115141335745](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115141335745.png)

# 2.优化案例

## 2.1.案例一:调整堆大小提升服务的吞吐量

调整堆大小服务的吞吐量会有不少的提升

## 2.2.案例二:逃逸分析

http://www.zhuguoqing.cn/?p=2789#17%E9%80%83%E9%80%B8%E5%88%86%E6%9E%90

## 2.3.案列三:合理配置堆内存

* 如果内存过大，那么如果产生FullGC的时候，GC时间会相对比较长，如果内存较小，那么就会频繁的触发GC;
* 实际上在内存很大的场景下,我们可以使用G1垃圾收集而不是使用推荐配置的大小;
* 如果内存不是很宽裕可以采用推荐配置进行设置大小;

### 2.3.1.推荐配置

![image-20211116092856830](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211116092856830.png)

* Java整个堆大小设置，Xmx 和 Xms设置为老年代存活对象的3-4倍，即FullGC之后的老年代内存占用的3-4倍。
* 方法区（永久代 PermSize和MaxPermSize 或 元空间 MetaspaceSize 和 MaxMetaspaceSize）设置为老年代存活对象的1.2-1.5倍;
* 年轻代Xmn的设置为老年代存活对象的1-1.5倍; 
* 老年代的内存大小设置为老年代存活对象的2-3倍;
* 上述官方推荐的配置只是一个参考;

### 2.3.2.计算老年代存活对象

* JVM参数中添加GC日志，GC日志中会记录每次FullGC之后各代的内存大小，观察老年代GC之后的空间大小。可观察一段时间内（比如2天）的FullGC之后的内存情况，根据多次的FullGC之后的老年代的空间大小数据来预估FullGC之后老年代的存活对象大小 ;
* 强制触发FullGC
  * `jmap -dump:live,format=b,file=heap.bin <pid>` 将当前的存活对象dump到文件，此时会触发FullGC;
  * `jmap -histo:live <pid>` 打印每个class的实例数目,内存占用,类全名信息.live子参数加上后,只统计活的对象数量. 此时会触发FullGC;
  * 在性能测试环境，可以通过Java监控工具来触发FullGC，比如使用VisualVM和JConsole，VisualVM集成了JConsole，VisualVM或者JConsole上面有一个触发GC的按钮;

### 2.3.3.UseAdaptiveSizePolicy

由于UseAdaptiveSizePolicy会动态调整 Eden、Survivor 的大小,有些情况存在Survivor 被自动调为很小，比如十几MB甚至几MB的可能，这个时候YGC回收掉 Eden区后，还存活的对象进入Survivor 装不下，就会直接晋升到老年代，导致老年代占用空间逐渐增加，从而触发FULL GC，如果一次FULL GC的耗时很长（比如到达几百毫秒），那么在要求高响应的系统就是不可取的;

**对于面向外部的大流量、低延迟系统，不建议启用此参数，建议关闭该参数;**

## 2.4.案例四:CPU占用很高的排查方案

1. `jps -lv`:查看所有Java进程;
2. `top -Hp <pid>`:根据进程ID检查使用异常线程的pid;

![image-20211117112440896](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211117112440896.png)

3. 把线程PID转化为16进制,`1465->0x5b9`;
4. 把线程信息打印出来`jstack <pid> > jstack.log`;
5. 查询jstack.log中的线程ID

![image-20211117112749135](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211117112749135.png)

## 2.5.案例五:G1并发执行的线程数对性能的影响

`-XX:ConcGCThreads=?`的配置堆性能会有不少的影响

## 2.6.案例六:调整垃圾回收器提高服务的吞吐量

关于SerialGC,ParallelGC,G1GC等在不同的系统配置下会有不同表现,可以先实际测一下再进行选择;

## 2.7.案例七:高并发订单系统JVM参数设置

![image-20211116232217525](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211116232217525.png)

我们可以使用G1垃圾收集,设置好` -XX:MaxGCPauseMillis`,`-XX:ConcGCThreads`;
