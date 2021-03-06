# JVM-2.垃圾收集器与内存分配策略

# 1.对象死亡判断

在堆中存放着Java几乎所有的对象实例,垃圾收集器在堆进行回收前,需要判断哪些对象还存活哪些对象已死.

## 1.1.引用计数法

在对象中添加一个引用计数器,每当有一个地方引用它时,计数器值就加一;当引用失效时,计数器值就减一;任何时刻计数器为零的对象就是不可能再被使用;在Java领域中并没有使用引用计数法,因为单纯的引用计数很难解决对象之间的相互循环依赖问题;

## 1.2.可达性分析算法

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210428173046503.png)

这个算法的基本思路就是通过一系列称为GC Roots的根对象作为起始节点集,从这些节点开始,根据引用关系向下搜索,搜索过程所走过的路径称为"引用链",如果某个对象到GC Roots间没有任何引用链项链,则证明此对象是不可能再被使用的.

java中,固定为GC Roots的对象包括以下:

* 在虚拟机栈(栈帧中的本地变量表)中引用的对象,比如各个线程被调用的方法堆栈中使用的参数,局部变量,临时变量;
* 方法区中类静态属性引用的对象,比如Java类的引用类型变量;
* 方法区中常量引用的对象,比如字符串常量池里的引用;
* 本地方法栈中JNI(Native方法)引用的对象;
* Java虚拟机内部的引用,如基本数据类型对应的Class对象,一些常驻的异常对象,还有系统类加载器;
* 所有被同步锁(synchronized关键字)持有的对象;
* 反应Java虚拟机内部情况的JMXBean,JVMTI中注册的回调,本地代码缓存等;

## 1.3.引用

JDK1.2.版本之后,Java对引用的概念进行了扩充,将引用分为强引用(St.rongly Reference),软引用(Soft Reference),弱引用(Weak Reference)和虚引用(Phantom Reference).这四种强度依次逐渐减弱;

* 强引用是最传统的"引用"的定义,是指在程序代码之中普遍存在的引用复制,即类似`Object obj = new Object()`.这种引用关系,无论任何情况下,只要强引用关系还存在,垃圾收集器就永远不会回收掉被引用的对象;
* 软引用是用来描述一些还有用,但非必须的对象.只是软引用关联着对象,在系统将要发生内存溢出异常前,会把这些对象列进回收范围之中进行第二次回收,如果这次回收还没有足够的内存,才会抛出内存溢出异常.JDK1.2提供`SoftReference`类来实现软引用;
* 弱引用是描述那些非必须对象,但是它的强度比软引用更弱一些,被弱引用关联的对象只能生存到下一次垃圾收集发生为止,当垃圾收集器开始工作的时候,无论当前内存是否足够,都会回收掉只被弱引用关联的对象.在JDK1.2版本之后提供`WeakReference`类来实现弱引用;
* 虚引用也称为幽灵引用或幻影引用,一个对象是否有虚拟引用的存在,完全不会对其生存时间构成影响,也无法通过虚引用来取得一个对象实例.为一个对象虚引用关联的唯一目的只是为了能在这个对象被收集器回收收到一个系统通知.在JDK1.2之后提供`PhatomReference`实现虚引用;

## 1.4.finalize()

要真正宣告一个对象死亡,至少要经历两次标记过程:

1. 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链,那它将会被第一次标记,随后进行一次筛选,筛选的条件是此对象是否有必要执行`finalize()`方法.假如对象没有覆盖`finalize()`方法,或者`finalize()`方法已经被虚拟机调用过,那么虚拟机将这两种情况都视为没有必要执行;如果这个对象被判定为确有必要执行`finalize()`方法,那么该对象将会被放置在一个叫做F-Queue的队列之中,并在稍后由一条虚拟机自动建立的低调度优先级的Finalizer线程去执行它们的的`finalize()`方法;
2. `finalize()`方法是对象逃脱死亡命运的的最后一次机会,稍后收集器将对F-Queue中的对象进行第二次小规模标记,如果对象要在`finalize()`中重新与应用链上的任何一个对象建立关联即可;

## 1.5.回收方法区

方法区的垃圾回收两部分内容:废弃的常量和不再使用的类型.回收废弃常量与回收Java堆中的对象非常类似.常量池中其他类(接口),方法,字段的符号引用也类似.

判断一个类型是否属于不再被使用的类的条件比较苛刻,同时满足三个条件:

1. 该类的所有实例都已被回收,也就是Java堆中不存在该类及其任何派生子类的实例;
2. 加载该类的类加载器已经被回收,一般很难达成;
3. 该类对应的Java.lang.Class对象没有在任何地方被引用,无法在任何地方通过反射访问该类的方法;

# 2.垃圾收集算法

垃圾收集算法可以划分为"引用计数式垃圾收集"(Reference Counting GC)和"追踪式垃圾收集"(Tracing GC)两大类,这两类也被称为"直接垃圾收集"和"间接垃圾收集",Java虚拟机采用的是追踪式垃圾收集.

## 2.1.分代收集

* 部分收集(Partial GC):指目标不是完整收集整个Java堆的垃圾收集,其中分为:
  * 新生代收集(Minor GC/Young GC):指目标只是新生代的垃圾收集;
  * 老年代收集(Major GC/Old GC):指目标只老年代的垃圾收集,目前只有CMS收集器单独收集老年代的行为;
  * 混合收集(Mixed GC):指目标是收集整个新声代以及部分老年代的垃圾收集,目前只有G1收集器会有这种;
* 整堆收集(Full GC):收集整个Java堆和方法区的垃圾收集;

## 2.2.标记-清除算法

首先标记出所有需要回收的对象,在标记完成后,统一回收掉被标记的对象,或者反过来,标记存活的对象,统一回收未被标记的对象;

## 2.3.标记-复制算法

Andrew Appel提出了一种优化过后的半区分带策略,HotSpot虚拟机的Serial,ParNew等新生代收集器均采用了这种策略来设计新生代的内存布局.

Appel式回收的具体做法是:把新生代分为一块较大的Eden空间和两块较小的Survivor空间,每次分配内存只使用Eden和其中一块Survivor空间,然后直接清理掉Eden和已使用过的那块Survivor空间.HotSpot虚拟机默认Eden和Survivor的大小比例是8:1,就是每次新生代可用内存为整个新生代容量的90%,只有一个Survivor空间,即10%的新生代是会被浪费的.当Survivor空间不足以容纳一次Minor GC之后存活的对象,就需要依赖其他的内存区域(实际上大多是老年代)进行分配和担保;

## 2.4.标记-整理算法

如果移动存活对象,尤其是在老年代这种每次回收都有大量对象存活区域,移动存活对象并更新所有应用这些对象的地方将会是一种极为负重的操作,而且这种对象移动的操作必须全程暂停用户应用程序才能进行,这种停顿被称为"Stop The World";

如果跟标记清除算法那样弥散于堆中的存活对象导致的空间碎片haul问题就只能依赖更为复杂的内存分配器和内存访问器来解决.比如通过"分区空闲分配链表"来解决内存分配问题.会影响应用程序的吞吐量.

HotSpot虚拟机里面关注吞吐量的Parallel Old收集器是基于标记-整理的,关注延迟CMS收集器是基于标记-清除算法的.CMS会暂时容忍内存碎片的存在,直到内存空间的碎片化程度已经大到影响对象的分配时,再采用标记-整理算法收集.

# 3.HotSpot的算法细节实现

目前功力不够,未能理解

# 4. 经典的垃圾收集器

![image-20210429101140691](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429101140691.png)

## 4.1.Serial收集器

使用标记-复制算法

这个收集器是一个单线程工作的收集器,它的单线程的意义并不仅仅是说明它只会使用一个处理器或一条收集线程去完成垃圾收集工作,更重要的是强调在它进行垃圾收集的时候,必须暂停其他所有工作线程,直到它收集结束.它依然是HotSpot虚拟机运行在客户端模式下的默认新生代收集器.

## 4.2.ParNew收集器

使用标记-复制算法

Serial的多线程并行版本.ParNew收集器是激活CMS`-XX:+UseConcMarkSweepGC`的默认新生代收集器,也可以使用`-XX:+/-UseParNewGC`选项强制指定或者禁用.

## 4.3.Parallel Scavenge收集器

使用标记-复制算法

Parallel Scavenge收集器的目标是达到一个可控制的吞吐量,所谓吞吐量是处理器用于运行用户代码的时间和处理器总耗时时间的比值.

* 停顿时间越短越适合需要与用户交互或需要保证服务质量的程序,良好的响应速度能提升用户体验;
* 高吞吐两则可以最高效率的利用处理器资源,尽快的完成程序的运算任务,主要适合用在后台运算不需要太多交互的任务;

`-XX:+UseAdativeSizePolicy`当这个参数被激活,虚拟机就会根据当前系统的运行情况收集性能监控信息,动态调整这些参数以提供最合适的停顿时间或者最大吞吐量.

## 4.4.Serial Old收集器

使用标记-整理算法

单线程工作.可以作为CMS收集器发生失败时的后备预案,在并发收集发生Concurrent Mode Failure时使用.

## 4.5.Parallel Old收集器

使用标记-整理算法

在注重吞吐量或者处理器资源较为稀缺的场合,都可以有限考虑Parallel Scavenge加Parallel Old收集器这个组合.

## 4.6.CMS收集器

使用标记-清除算法

CMS(Concurrent Mark Sweep)收集器是一种以获取最短停顿时间为目标的收集器,整个过程需分为4个步骤

1. 初始标记(CMS initial mark)*需要STW*;初始标记仅仅标记GC Roots能关联到的对象,速度很快;
2. 并发标记(CMS concurrent mark);并发标记就是从GC Roots的直接关联对象开始遍历整个对象图的过程,可以与垃圾收集线程一起并发运行;
3. 重新标记(CMS remark)*需要STW*;重新标记是为了修正并发标记期间,因用户继续运作而导致标记产生变动的那一部分对象的标记记录;
4. 并发清除(CMS concurrent sweep);并发清除,由于不需要移动存活对象,所以这个阶段也是可以与用户线程同时并发的;

![image-20210429105622438](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429105622438.png)

CMS处理器对处理器资源非常敏感,在并发阶段,虽然不会导致用户线程停顿,但会因为占用了一部分的线程而导致应用程序变慢,降低总吞吐量.CMS收集器无法处理"浮动垃圾"(Floating Garbage),有可能出现"Concurrent Mode Failure"失败进而导致另一次的Stop the world的Full GC的产生.在CMS的并发标记和并发清理阶段,这一部分垃圾对象是出现在标记过程结束后,CMS无法在当次收集中集中处理掉他们,只能留待下一次垃圾收集时再清理掉,这部分叫做浮动垃圾.所以CMS必须预留一部分空间供并发收集使用.`-XX:CMSInitiatingOccu-pancyFraction`的值来提高CMS的触发百分比.要是CMS运行期间预留的内存无法满足程序分配新对象的需要,就会出现并发失败,这时候后启动后备预案:冻结用户线程的执行,临时启用Serial Old收集器来重新进行老年代的垃圾收集,但这样停顿时间就很长.

CMS是基于标记-清除算法实现的收集器,如果碎片过多,给大对象分配带来很大的麻烦,不得不触发一次Full GC,虚拟机提供了一些参数在Full GC进行碎片整理.

## 4.7.G1收集器

G1 开创的是基于Region的堆内存布局.把连续的Java堆划分为多个大小相等的独立区域(Region),每个Region都可以根据需要,扮演新生带的Eden空间,Survivor空间,或者老年代空间.收集器能够扮演不同的角色的Region采用不同的策略去处理.Region中还有一类特殊的Humongous区域,专门用来存储大对象.G1认为只要大小超过一个Region容量一半的对象即可判定为大对象.每个Region的大小可以通过参数`-XX:G1HeapRegionSize`设定,取值范围为1MB-32MB.且应为2的N次幂.而对于那些超过了整个Region容量的超大对象,将会被放在N个连续的Humongous Region之中.G1大多数行为把Humongous Region作为老年代的一部分开看待.

虽然G1仍然保留新生代和老年代的概念,但新生代和老年代不再是固定的.它们都是一些列区域(不需要连续)的动态几何.G1收集器之所以能建立可预测的停顿时间模型,是因为它将Region作为单次回收的最小单元,即每次收集到的内存空间都是Region大小的整数倍.这样可以有计划的避免在整个Java堆中进行全区域的垃圾收集.更具体的处理思路是让G1收集器去跟踪各个Region里面的垃圾堆积的价值大小,价值即回收所获得的空间大小以及回收所需时间的经验值.然后在后台维护一个优先级列表,每次根据用户设定允许的收集停顿时间(使用参数`-XX:MaxGCPauseMillis`指定,默认值是200毫秒),优先处理回收价值收益直达的Region.

# 5.选择合适的垃圾收集器

## 5.1.收集器的权衡

看自己的选择

## 5.2.虚拟机及垃圾收集器日志

在JDK9以前,HotSpot并没有提供统一的日志处理框架,虚拟机各个功能模块的日志开关分布在不同的参数上,日志级别,循环日志大小,输出格式,重定向等设置在不同功能上都要单独解决.到了JDK9,HotSpot所有功能都收归到了`-Xlog`参数上.

1. 查看GC基本信息,在JDK9之前使用`-XX:+PrintGC`,JDK后使用`-Xlog:gc:`;下图是JDK8

![image-20210429142120500](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429142120500.png)

2. 查看GC详细信息,在JDK9之前使用`-XX:+PrintGCDetails`,在JDK9后使用`-X-log:gc*`用通配符将GC标签下所有细分过程都打印出来;下图是JDK8;

![image-20210429142407191](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429142407191.png)

3. 查看GC前后的堆,方法区可用容量变化,在JDK9之前使用`-XX:+PrintHeapAtGC`,JDK9之后使用`-Xlog:gc+heap=debug`;

![image-20210429142728923](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429142728923.png)

4. 查看GC过程中用户线程并发时间以及停顿时间,在JDK9之前使用`-XX:+PrintGCApplicationConcurrentTime`以及`-XX:+PrintGCApplicationStoppedTime`,在JDK9之后`-Xlog:safepoint`;

![image-20210429143025676](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429143025676.png)

5. 查看收集器Ergonomics机制(自动设置堆空间各分代区域大小,收集目标等内容,从Parallel收集器开始支持)自动调节的相关信息.在JDK9之前使用`-XX:+PrintAdaptiveSizePolicy`,JDK9之后使用`-Xlog:gc+ergo*=trace`;

![image-20210429143428321](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429143428321.png)

6. 查看熬过收集后剩余对象的年龄分布信息,在JDK9之前使用`-XX:+PrintTenuringDistribution`,JDK9之后使用`-Xlog:gc+age=true`;

![image-20210429143744396](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429143744396.png)

下图是GC的日志参数:

![image-20210429143934329](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429143934329.png)

![image-20210429144016012](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429144016012.png)

![image-20210429144311928](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429144311928.png)

![image-20210429144327211](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210429144327211.png)

# 6.内存分配与回收策略

本次的内存分配与回收策略的前提是:HotSpot虚拟机,以客户端模式运行,使用Serial+Serial Old客户端默认收集器组合下的内存分配和回收策略.

* 大多数情况下,对象在新生代Eden去中分配,当Eden去没有足够空间进行分配时,虚拟机将发起一次Minor GC;
* 大对象直接进入老年代,大对象指的就是需要大量连续内存空间的Java对象,最典型的大对象就是很长的字符串,或者元素数据很庞大的数组;
* 长期存活的对象将进入老年代.虚拟机给每个对象定义了一个对象年龄(Age)计数器,存储在对象头中.对象每熬过一次Minor GC年龄就加一岁,当年龄增加到一定程度(默认15),就会被晋升到老年代中.对于晋升老年代的年龄阈值,可以通过参数`-XX:MaxTenuringThreshold`设置;
* 动态对象年龄判定,如果在Survivor空间中低于或等于某年龄的所有对象大小的总和大于Survivor空间的一般,年龄大于或等于该年龄的对象就可以直接进入老年代;
* 空间分配担保,在发生Minor GC之前,虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间,如果这个条件成立,那这一次Minor GC可以确保是安全的.如果不成立,则虚拟机会先查看`-XX:HandlePromotionFailure`参数的设置值是否允许担保失败,如果允许,那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小,如果大于,尝试进行一次Minor GC,如果小于,或者`-XX:HandlePromotionFailure`设置不允许毛线,改为进行一次Full GC;JDK6 Update24后的规则是只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小,就会进行Minor GC,否则进行Full GC;



