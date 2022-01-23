# JVM详解05

# 1.垃圾回收器概述

## 1.1.垃圾回收器的分类

* 按照线程数分,可以分为串行垃圾回收器和并行垃圾回收器
  * 串行回收指的是在同一时间段内只允许有一个CPU用于执行垃圾回收操作,此时工作线程被暂停,直到垃圾收集工作结束;client模式下的JVM就是串行回收;
  * 并行回收可以运用多个CPU同时执行垃圾回收,因此提升了应用的吞吐量,不过并行回收仍然与串行回收一样,采用独占式,会STW;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827203727964.png" alt="image-20210827203727964" style="zoom: 50%;" />

* 按照工作模式分,可以分为并发式垃圾回收和独占式垃圾回收;

  * 并发式垃圾回收器与应用程序线程交替工作,以尽可能减少应用程序的停顿时间;
  * 独占式垃圾回收器一旦运行,就停止应用程序中的所有用户线程,直到垃圾回收过程完全结束;

  <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827204300662.png" alt="image-20210827204300662" style="zoom: 67%;" />

## 1.2.评估GC的性能指标

* **吞吐量:运行用户代码的时间占总运行时间(总运行时间=程序的运行时间+内存回收的时间)的比例;**
* 垃圾收集开销:吞吐量的补数,垃圾收集所占用时间与总运行时间的比例;
* **暂停时间(延迟):执行垃圾收集时候,程序的工作线程被暂停的时间;**
* 收集频率:相对于应用程序的执行,收集操作发生的频率;
* **内存占用:Java堆区所占的内存大小;** 

吞吐量,暂停时间,内存占用三者共同构成一个不可能三角.

**这三项中,暂停时间的重要性日益凸显.因为随着硬件发展,内存占用多些越来越能容忍,硬件性能的提升也有助于降低收集器运行时对应用程序的影响,即提高了吞吐量.而内存的扩大,对延迟反而带来了负面效果**

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827223931205.png)

吞吐量与暂停时间:

* 高吞吐量较好因为这会让应用程序的最终用户感觉只有应用程序线程在做生产性工作;
* 低暂停时间较好因为从最终用户的角度来看不管是GC还是其他原因导致一个应用被挂起始终是不好的.对于交互式应用程序来说,需要低暂停时间; 

现在标准:**在停顿时间可控(比如10ms以内)的标准下,提高吞吐量**;

## 1.3.不同的垃圾收集器

* 串行回收器:Serial,Serial Old;
* 并行回收器:ParNew,Parallel Scavenge,Parallel Old;
* 并发回收器:CMS,G1;

![image-20210827232256545](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827232256545.png)

* Serial Old作为CMS出现`Concurrent Mode Failure`失败的后备预案;
* 红色虚线:在JDK 8时,将Serial+CMS,ParNew+Serial Old这两个组合声明废弃,并在JDK 9中完全取消了这些组合;
* 绿色虚线:在JDK 14中,弃用了Parallel Scavenge和SerialOld GC组合;
* 青色虚线:在JDK 14中,删除CMS垃圾回收器;

`-XX:+PrintCommandLineFlags`:查看命令行相关参数(包含使用的垃圾收集器);

`jinfo -flag [相关垃圾回收器参数如UseParallelGC] [进程ID]`;

# 2.Serial GC

* Serial收集器采用**复制算法**,串行回收和STW机制的方式执行内存回收;
* Serial Old收集器同样采用了串行回收和STW机制,只不过内存回收算法使用的**标记压缩**算法;
* Serial Old主要有两个用途:1.与新生代Parallel Scavenge配合,2.作为老年代CMS收集器的后备垃圾收集方案;
* HotSpot虚拟机中,使用`-XX:+UseSerialGC`参数可以指定年轻代和老年代都使用串行收集器;

![image-20210828110741251](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828110741251.png)

* 优势是简单高效,对于限定单个CPU的环境来说,Serial收集器由于没有线程交互的开销,专心做垃圾收集自然可以获得最高的单线程收集效率;

# 3.ParNew GC

* ParNew收集器除了采用并行回收的方式执行内存回收外,与Serial GC之间几乎没有区别.ParNew收集器在年轻代中同样也是采用复制算法,STW机制;
* 使用`-XX:+UseParNewGC`手动指定使用ParNew收集器执行内存回收任务.表示年轻代使用并行收集器,不影响老年代;
* `-XX:ParallelGCThreads`限制线程数量,默认开启和CPU数据相同的线程数;

![image-20210828154452862](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828154452862.png)

# 4.Parallel GC(高吞吐)

* Parallel Scavenge GC同样采用了复制算法,并行回收和STW机制;
* Parallel Scavenge收集器的目标是达到一个**可控制的吞吐量**,也被称为吞吐量优先的垃圾收集器;
* Parallel Scavenge的自适应调节策略也是Parallel Scavenge与ParNew的一个区别;
* Parallel Old收集器采用了标记压缩算法,同样也是基于并行回收和STW;

![image-20210828161510814](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828161510814.png)

* `-XX:+UseParallelGC`:手动指定年轻代使用Parallel并行收集器执行内存回收任务;
* `-XX:+UseParallelOldGC`:手动指定老年代使用Parallel Old收集器;
* 注意:上述两个参数,默认开启一个,另一个也会被开启,互相激活;
* `-XX:+ParallelGCThreads`设置年轻代并行收集器的线程数.一般地最好与CPU数量相等,避免过多的线程数影响垃圾收集性能;
  * 默认情况下,CPU数量小于等于8个,ParallelGCThreads的值等于CPU数量;
  * 当CPU数量大于8个,ParallelGCThreads的值是3+(5*cpu)/8;
* `-XX:MaxGCPauseMillis`:设置垃圾收集器最大暂停时间,单位是毫秒;
  * 为了尽可能把停顿时间控制在MaxGCPauseMills以内,收集器在工作时会调整Java堆大小或者其他参数;该参数慎用;
* `-XX:GCTimeRatio`:设置吞吐量,垃圾收集时间占总时间的比例( = 1/(N+1));
  * 取值范围(0,100),默认值99,也就是垃圾回收时间不超过1%;
  * 这个参数与前一个`-XX:MaxGCPauseMillis`参数有一定矛盾,暂停时间越长,Radio参数就容易超过设定比例;
* `-XX:+UseAdaptiveSizePolicy`:设置Parallel Scavenge收集器具有自适应调节策略;默认开启;
  * 这种模式下,年轻代的大小,Eden和Survivor的比例,晋升老年代的对象年龄等参数会被自动调整,已达到在堆大小,吞吐量和停顿时间之间的平衡点;
  * 在手动调优比较困难的场合,可以直接使用这种方式,仅指定虚拟机的最大堆,目标吞吐量和停顿时间,让虚拟机自己完成调优工作;

# 5.CMS GC(低延迟)

CMS关注的点事尽可能缩短垃圾收集时用户线程的停顿时间.停顿时间越短就越适合于用户交互的程序,良好的响应速度能提升用户体验;目前很大一部分的Java应用集中在互联网站或者B/S系统的服务端上,这类应用尤其重视服务的响应速度,希望系统停顿时间最短,以给用户带来较好的体验,CMS收集器就非常符合这类应用需求;CMS采用的是**标记-清除**算法,并且也会STW;

![image-20210828172518890](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828172518890.png)

* 初始标记:在这个阶段中,程序中所有的工作线程都会因为STW机制出现短暂的暂停,这个阶段的主要任务仅仅是**标记出GC Roots能直接关联到的对象**,一旦标记完成后就会恢复之前被暂停的所有应用线程.由于直接关联对象比较小,所以这里的速度非常快;
* 并发标记:**从GC Roots的直接关联对象开始遍历整个对象图的过程**,这个过程耗时较长但是不需要停顿用户线程;
* 重新标记:由于在并发标记解读啦,程序的工作线程会和垃圾收集线程同时运行或者交叉运行,因此**为了修正并发标记阶段,因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录**,这个阶段的停顿时间通常比初始标记阶段稍微长一些,但远比并发标记阶段的时间短;
* 并发清除:此阶段清理删除掉标记阶段判断的已经死亡的对象,释放内存空间,由于不需要移动存活对象,所以这个阶段也是可以与用户线程同时并发的;

CMS GC的优缺点:

* 由于最耗费时间的并发标记和并发清除都不需要暂停工作,所以整体的回收是低停顿的;
* 由于在垃圾收集阶段用户线程没有中断,所以在CMS回收过程中,还应该确保应用程序用户线程有足够的内存空间可用;因此,CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集,而是当堆内存使用率达到某一阈值时,便开始进行回收,以确保应用程序在CMS工作过程依然有足够的空间支持应用程序运行.要是CMS运行期间预留的内存无法满足程序需要,就会出现一次Concurrent Mode Failure失败,这时虚拟机将启动后备预案:临时启动Serial Old收集器来重新进行老年代的垃圾收集,这样停顿的时间就很长了;
* CMS收集器的垃圾收集算法采用的是标记清除算法,会产生内存碎片,需要维护空闲列表进行内存分配;并发清除后,用户线程可用空间可能不足,在无法分配大对象情况下,提前Full GC;
* CMS的重新标记阶段只是为了取消掉一开始是垃圾后来不是垃圾的阶段,在并发标记阶段如果产生新的垃圾对象,CMS是无法对这些垃圾对象进行标记的,最终会导致这些新产生的垃圾对象没有被及时回收,这就是浮动垃圾;

CMS GC的参数:

* `-XX:+UseConcMarkSweepGC`手动指定使用CMS收集器执行内存回收任务;
  * 开启后该参数会自动将`-XX:+UseParNewGC`打开.即:ParNew+CMS+Serial Old的组合;
* `-XX:CMSInitiatingOccupanyFraction`设置堆内存使用率的阈值,一旦达到该阈值,便开始进行回收;
  * JDK5之前版本的默认值是68,JDK6及以上版本默认值是92,即当老年代的空间使用率到达92%时执行CMS回收;
  * 如果内存增长缓慢,可以设置一个稍大的值,大的阈值可以有效降低CMS的触发频率,减少老年代回收的次数可以较为明显地改善应用程序性能.反之,如果应用程序内存使用率增长很快,则应该降低这个阈值,以避免频繁触发老年代串行收集器.通过该选项可以邮箱降低Full GC的执行次数;
* `-XX:+UseCMSCompactAtFullCollection`:用于指定在执行FullGC后对内存空间进行压缩整理,以此避免内存碎片的产生,不过由于内存压缩整理过程无法并发执行,带来的问题就是停顿时间变得长了;
* `-XX:CMSFullGCBeforeCompaction`:设置在执行多少次Full GC后对内存空间进行压缩整理;
* `-XX:ParallelCMSThreads`:设置CMS的线程数量;
  * CMS默认启动的线程数是(ParallelGCThreads+3)/4,ParallelGCThreads是年轻代并行收集器的线程数.当CPU资源比较紧张时候,受到CMS收集器线程的影响应用程序的性能在垃圾回收阶段可能会非常糟糕;

# 6.G1 GC(低延迟)

官方给G1设定的目标是在延迟可控的情况下获得尽可能高的吞吐量;

G1是一个并行回收器,它把堆内存分割为很多不想关的区域.使用不同的Region来表示Eden,Survivor0,Survivor1,Old;G1有计划地避免在整个Java堆中进行全区域的垃圾收集.**G1跟踪各个Region里面的垃圾堆的价值大小(回收所获得的空间大小以及回收所需要时间的经验值),在后台维护一个优先列表,每次根据允许的收集时间,优先回收价值最大的Region**;

考虑到G1只回收一部分Region,停顿时间是用户可控制的,所以并没有实现回收阶段设计成与用户程序一起并发执行,而是把这个特性放到了G1之后的低延迟垃圾收集ZGC.另外考虑到G1不仅面向低延迟还面向高吞吐量所以选择了完全暂停用户线程的实现方案;

## 6.1.G1 GC优点

![image-20210828193725365](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828193725365.png)

* 并行与并发
  * 并行性:G1在回收期间,可以有多个GC线程同时工作,有效利用多核计算能力.此时用户线程STW;
  * 并发性:G1拥有与引用程序交替执行的能力,部分工作可以与应用程序同时执行,因此,一般来说,不会在整个回收阶段发生完全阻塞应用程序的情况;
* 分代收集
  * G1依然属于分代型垃圾回收器,它会区分年轻代和老年代,它将堆空间分为若干个区域Region,这些区域包含了逻辑上的年轻代和老年代;
* 空间整合
  * G1将内存划分为一个个的Region,内存回收是以region作为基本单位的,Region之间是**复制算法**,但整体上实际可以看作是**标记压缩**算法,两种算法都可以避免内存碎片.

* 可预测的停顿时间模型(即软实时 soft real-time)
  * G1除了追求低停顿外,还能建立可预测的停顿时间模型,能让使用者明确指定在一个长度为M毫秒的时间片段,好小在垃圾收集上的时间不得超过N毫秒;
  * 由于分区原因,G1可以只选取部分区域进行内存回收,这样缩小了回收的范围,对于全局停顿情况的发生也能得到好的控制;
  * **G1跟踪各个Region里面的垃圾堆积的价值大小,在后台维护一个优先级列表,每次根据允许的收集时间,优先回收价值最大的Region,保证了G1收集器在有限时间内可以获得尽可能高的收集效率**;
* HotSpot垃圾收集器中,除了G1以外,其他的垃圾收起使用内置的JVM线程执行GC的多线程操作,而G1 GC可以采用应用线程承担后台运行的GC工作,即当JVM的GC线程处理速度慢时,系统会调用应用程序线程帮助加速垃圾回收过程;

## 6.2.G1 GC缺点

* G1无论是为了垃圾收集产生的内存占用(Footprint)还是程序运行时的额外执行负载(Overload)都要比CMS要高;
* 在小内存引用上CMS的表现大概率会优于G1,G1在大内存应用上发挥其优势,平衡点是6-8GB;

## 6.3.G1 GC的参数

* `-XX:+UseG1GC`:手动指定使用G1收集器执行内存回收任务;
* `-XX:G1HeapRegionSize`:设置每个Region的大小,值是2的幂,范围是1MB-32MB之间,目标是根据最小的Java堆大小划分出约2048个区域,默认是堆内存的1/2000;
* `-XX:MaxGCPauseMillis`:设置期望达到的最大GC停顿时间指标,JVM不保证达到,默认200ms;
* `-XX:ParallelGCThread`:设置STW时GC工作线程数,最多为8;
* `-XX:ConcGCThreads`:设置并发标记的线程数.将n设置为并行垃圾回收线程数(ParallelGCThreads)的1/4左右;
* `-XX:InitiatingHeapOccupancyPercent`:设置触发并发GC周期的Java堆占用率阈值,超过此值,触发GC,默认45;

G1的设计原则就是简化JVM性能调优,我们只需要简单3步就可以完成调优;

1. 开启G1垃圾收集器;
2. 设置堆的最大内存;
3. 设置最大停顿时间;

## 6.4.Remembered Set

* 一个对象被不同区域引用;一个Region不可能是孤立的,一个Region中的对象可能被其他任意Region中对象引用;判断对象存活时候的时候,是否需要扫描整个Java堆才能保证准确?
* 解决方法:
  * 无论G1还是其他分代收集器,JVM都是使用Remembered Set来避免全局扫描;
  * 每个Region有一个对应的Remembered Set;
  * 每次Reference类型数据写操作时候,都会产生一个Write Barrier暂时中断操作;
  * 然后检查将要写入的引用指向的对象是否和该Reference类型数据在不同的Region;
  * 如果不同,通过CardTable把相关引用信息记录到引用指向对象的所在Region对应的Remembered Set中;
  * 当进行垃圾收集时,在GC根节点的枚举范围加入Remembered Set:这样就可以保证不进行全局扫描;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828222430985.png" alt="image-20210828222430985" style="zoom: 33%;" />

## 6.5.G1 GC的过程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828215932876.png" alt="image-20210828215932876" style="zoom:50%;" />

G1 GC的过程为如下四个环节

* Young GC;
* Concurrent Marking;老年代并发标记;
* Mixed GC;混合回收;
* 如果需要,单线程,独占式,高强度的Full GC还是继续存在的,它针对GC的评估失败提供了一种失败保护机制,即强力回收;

> * 应用程序分配内存,当年轻代的Eden区用尽时开始年轻代回收过程;G1的年轻代收集阶段是一个并行的独占式收集器,在年轻代回收期,G1 GC暂停所有应用程序线程,启动多线程执行年轻代回收.然后从年轻代区间移动存活对象到Survivor区间或者老年区间,也有可能是两个区间都会涉及;
>
> * 当堆内存使用达到一定值时候(默认45%),开始老年代并发标记;
> * 标记完成后开始混合回收过程,G1 GC从老年区间移动存活对象到空闲区间,这些空闲区间也就成为了老年代的一部分,和年轻代不同,老年代的G1回收器和其他GC不同,G1的老年代回收器不需要整个老年代被回收,一次只需要扫描回收一小部分老年代的Region就可以了.同时这个老年代Region是和年轻代一起被回收的;

例:一个Web服务器,Java进程最大堆内存4G,每分钟响应1500个请求,每45秒回新分配2G内存,G1会每45秒进行一次年轻代回收.每31小时整个堆的使用率会达到45%,会开始老年代并发标记过程,标记完成后开始四到五次的混合回收;

### 6.5.1.G1 Young GC

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210828222943609.png" alt="image-20210828222943609" style="zoom:50%;" />

1. 扫描根:根是指static变量指向的对象,正在执行的方法调用链上的局部变量等;根引用连同RSet记录的外部引用作为扫描存活对象的入口;
2. 更新RSet:处理dirty card queue中的card,更新RSet,此阶段完成后,RSet可以准确的反应老年代对所在内存分段中对象的引用;
3. 处理RSet:识别被老年代对象指向的Eden中的对象,这些被指向的Eden中的对象被认为是存活的;
4. 复制对象:对象树被遍历,Eden区存段中存活的对象会被复制到Survivor区中空的内存分段,Survovor区内存段中存活的对象如果年龄未达到阈值,年龄加1,达到阈值会被复制到Old区中空的内存分段.如果Survivor空间不够,Eden空间的部分数据会直接晋升到老年代空间;
5. 处理引用:处理Soft,Weak,Phantom,Final,JNI Weak等引用,最终Eden空间的数据为空,GC停止,而目标内存的对象都是连续存储的,没有碎片,所以复制过程可以达到内存整理的效果,减少碎片;

### 6.5.2.G1 Concurrent Marking

1. 初始标记:标记从根节点直接可达的对象,这个阶段是STW的,并且会触发一次年轻代GC;
2. 根区域扫描:G1 GC扫描Survivor区直接可达的老年代区域对象,并标记被引用的对象.这一过程必须在Young GC之前完成;
3. 并发标记:在整个堆中进行并发标记,此过程可能被Young GC中断,在并发标记阶段,如果发现区域对象中的所有对象都是垃圾,那么这个区域会被立即回收.同时,并发标记过程中,会计算每个区域的对象活性(区域中存活对象的比例);
4. 再次标记:由于应用程序持续进行,需要修正上一次标记的结果,是STW的,G1采用了比CMS更快的初始快照算法:snapshot-at-the-beginning(SATB);
5. 独占清理:计算各区域的存活对象和GC回收比例,并进行排序,识别可以混合回收的区域,为下阶段做铺垫,是STW的;这个阶段并不会实际上去做垃圾的收集;
6. 并发清理阶段:识别并清理完全空闲的区域;

### 6.5.2.G1 Mixed GC

当越来越多的对象晋升到老年代old region时,为了避免堆内存被耗尽,虚拟机会触发一个混合的垃圾收集器;即Mixed GC,该算法会回收整个Young Region还会回收一部分Old Region.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829010500700.png" alt="image-20210829010500700" style="zoom:50%;" />

1. 并发标记结束后,老年代中百分百为垃圾的内存分段被回收了,部分垃圾的内存分段被计算出来了.默认情况下,老年代的内存分段会分8次(可以设置`-XX:G1MixedGCCountTarget`设置)被回收;
2. 混合回收的回收集(Collection Set)包括八分之一的老年代内存分段,Eden区内存分段,Survivor去内存分段,混合回收的算法与年轻代回收算法一样,只是回收集多了老年代的内存分段而已;
3. 由于老年代的内存分段默认分8次,G1会优先回收垃圾多的内存分段.垃圾占内存分段比例越高,越先被回收.并且由一个阈值会决定内存分段是否被回收;`-XX:G1MixedGCLiveThresholdPercent`默认65%,意思垃圾占内存分段要达到65%才会回收,如果垃圾占比太低,意味着存活对象占比高复制的时候花更多的时间;
4. 混合回收并不一定要进行8次,有`-XX:G1HeapWastePercent`,默认值是10%,意思是允许整个堆中有10%的空间被浪费,意味着如果发现可以回收的垃圾比例低于10%,不再进行混合回收;

### 6.5.3.G1 Full GC

G1的初衷是避免Full GC的出现,如果上述方式不能正常工作,G1会STW,使用单线程的内存回收算法进行垃圾收集,性能很差;

导致G1 Full GC的原因可能有两个:

1. Evacuation的时候没有足够to-space来存放晋升对象;
2. 并发处理过程完成之前空间耗尽;

## 6.6.G1 GC优化

* 年轻代大小:
  * 避免使用`-Xmn`或`-XX:NewRatio`等相关选项显式设置年轻代大小;
  * 固定年轻代的大小会覆盖暂停时间目标;
* 暂停时间目标不要太过严苛
  * G1 GC的吞吐量目标是90%的应用程序时间和10%的垃圾回收时间;
  * 暂停时间目标太严苛会直接影响到吞吐量;

# 7.经典垃圾回收器的对比

![image-20210829013549513](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829013549513.png)

# 8.GC日志分析

## 8.1.GC日志的参数列表

* `-XX:+PrintGC`:输出GC日志.类似`-verbose:gc`;
* `-XX:+PrintGCDetails`:输出GC详细日志;
* `-XX:+PrintGCTimeStamps`:输出GC的时间戳(以基准时间的形式);
* `-XX:+PrintGCDateStamps`:输出GC的时间戳(以日期的形式);
* `-XX:+PrintHeapAtGC`:在进行GC的前后打印出堆的信息;
* `-Xloggc:../logs/gc.log`:日志文件的输出路径;

## 8.2.GC日志的说明

`-XX:+PrintGCDetails`

~~~shell
6.854: [GC (Allocation Failure) [PSYoungGen: 16294K->2032K(18432K)] 16294K->13938K(59392K), 0.0068172 secs] [Times: user=0.01 sys=0.02, real=0.01 secs] 
# user:指的是垃圾收集器花费的所有CPU时间;
# sys:花费在等待系统调用或系统事件的时间;
# real:GC从开始到结束的时间,包括其他进程占用时间片的实际时间;由于多核的原因,user时间可能会超过real时间;
# Serial的新生代名字是Default New Generation,显示DefNew; Serial新生代名字是Parallel New Generation显示ParNew; Parallel显示PSYoungGen;G1显示garbaage-first-heap;
15.175: [GC (Allocation Failure) [PSYoungGen: 18416K->2016K(18432K)] 30322K->30220K(59392K), 0.0078215 secs] [Times: user=0.01 sys=0.02, real=0.01 secs] 
15.183: [Full GC (Ergonomics) [PSYoungGen: 2016K->0K(18432K)] [ParOldGen: 28204K->30032K(40960K)] 30220K->30032K(59392K), [Metaspace: 3587K->3587K(1056768K)], 0.0130318 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
# 中括号内是各自区域的回收前占用大小,回收后占用大小,以及区域总大小;
# 中括号外是年轻代和老年代的总大小;
23.569: [Full GC (Ergonomics) [PSYoungGen: 16314K->5500K(18432K)] [ParOldGen: 30032K->40835K(40960K)] 46347K->46335K(59392K), [Metaspace: 3587K->3587K(1056768K)], 0.0086419 secs] [Times: user=0.02 sys=0.01, real=0.01 secs]

Heap
 PSYoungGen      total 18432K, used 10080K [0x00000007bec00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 16384K, 61% used [0x00000007bec00000,0x00000007bf5d8248,0x00000007bfc00000)
  from space 2048K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007c0000000)
  to   space 2048K, 0% used [0x00000007bfc00000,0x00000007bfc00000,0x00000007bfe00000)
 ParOldGen       total 40960K, used 40835K [0x00000007bc400000, 0x00000007bec00000, 0x00000007bec00000)
  object space 40960K, 99% used [0x00000007bc400000,0x00000007bebe0c20,0x00000007bec00000)
 Metaspace       used 3594K, capacity 4536K, committed 4864K, reserved 1056768K
  class space    used 398K, capacity 428K, committed 512K, reserved 1048576K
~~~

![image-20210829102447252](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829102447252.png)

![image-20210829102712596](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829102712596.png)

## 8.3.日志分析工具

常用的日志分析工具:GCViewer,GCEasy,GCHisto,GCLogViewer,Hpjmeter,garbagecat等;

# 9.Shenandoah GC

Open JDK12引入Shenandoah GC

Shenandoah GC最初由RedHat进行一项垃圾收集器研究项目Pauseless GC的实现,用于针对JVM上的内存回收实现低停顿需求,在2014年贡献给OpenJDK;

![image-20210829141810631](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829141810631.png)

上图是Shenandoah开发团队在实际应用中的测试数据;

# 10.ZGC

ZGC收集器是一块基于Region内存布局的,不设分代的,使用了读屏障,染色指针和内存多重映射等技术实现**可并发标记-压缩算法**的,以低延迟为首要目标的一款垃圾收集器;

ZGC的工作过程分为4个阶段:

1. 并发标记;
2. 并发预备重分配;
3. 并发重分配;
4. 并发重映射;

ZGC几乎在所有地方并发执行的,除了初始标记是STW.所以停顿时间几乎就耗费在初始标记上,这部分的实际时间是非常少的;

![image-20210829142718123](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210829142718123.png)

