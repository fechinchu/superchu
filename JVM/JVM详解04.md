# JVM详解04

# 1.垃圾回收简介

在早期的C/C++时代,垃圾回收基本上是手工进行的.开发人员可以使用new关键字进行申请.并使用delete关键字进行内存释放.

```c++
MibBride *pBridge = new cmBaseGroupBride();
delete pBridge;
```

这种方式可以灵活控制内存释放的时间,但是会给开发人员带来频繁申请和释放内存的管理负担.倘若有一处内存区间由于程序员编码问题忘记被回收,那么就会产生**内存泄漏**;垃圾对象永远无法被清除,随着系统运行时间的不断增长,垃圾对象所消耗的内存可能持续上升,直到出现**内存溢出**并造成引用程序崩溃;

# 2.垃圾回收算法

## 2.1.垃圾标记阶段-引用计数算法

* 引用计数算法,对每个对象保存一个整型的引用计数器属性.用于记录对象被引用的情况;

* 对于一个对象A,只要有任何一个对象引用了A,则A的引用计数器就加1;当引用失效时候,引用计数器就减1;只要对象A的引用计数器的值为0;即表示对象A不可能再被使用,可进行回收;
* 优点:
  * 实现简单,垃圾对象便于辨识,判定效率高;回收没有延迟;
* 缺点:
  * 需要单独的字段存储计数器,这样的做法增加了存储空间的开销;
  * 每次复制都需要更新计数器,伴随着加法和减法的操作,增加了时间开销;
  * 引用计数器无法处理循环引用的情况,这条导致在Java垃圾回收器中没有适用这类算法;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210822135729567.png" alt="image-20210822135729567" style="zoom: 67%;" />

* python同时支持引用计数和垃圾收集机制;
* python如何解决循环引用?
  * 手动解除:在合适的实际,解除引用关系;
  * 使用弱引用weakref,weakref是python提供的标准库,用来解决循环引用;

 ## 2.2.垃圾标记阶段-可达性分析算法

* 相对于引用计数算法而言,可达性分析算法不仅同样具备实现简单和执行高效等特点,更重要的是该算法可以有效**解决在引用计数算法中循环引用的问题,房子内存泄漏的发生**;

* 相较于引用计数算法,可达性分析算法是Java,C#选择的.这种类型的垃圾收集通常叫做**追踪新垃圾收集**;

### 2.2.1.可达性分析

* GC Roots根集合就是一组必须活跃的引用;
* 可达性分析算法是以根对象集合(GC Roots)为起始点,按照从上而下的方式**搜索被根对象集合所连接的目标对象是否可达**;
* 内存中的存活对象都会被根对象集合直接或间接连接着,搜索所走过的路径被称为引用链(Reference Chain);
* 如果目标对象没有任何引用链相连,则是不可达的,就意味着该对象已经死亡,可以标记为垃圾对象;
* 在可达性分析算法中,只有能够被根对象集合直接或者间接连接的对象才是存活的对象;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210822145805498.png" alt="image-20210822145805498" style="zoom:50%;" />

 ### 2.2.2.GC Roots

1. 虚拟机栈中引用的对象;

2. 比如:各个线程被调用的方法中使用到的参数,局部变量等;

3. 本地方法栈内JNI(本地方法)引用的对象;

4. 类静态属性引用的对象;比如:Java类的引用类型静态变量;

5. 常量对应的对象;比如:字符串常量池(String Table)里的引用;

6. 所有被同步锁synchronized持有的对象;

7. Java虚拟机内部引用

8. 基本数据类型对应的Class对象,一些常驻的异常对象(如:NullPointerException,OutOfMemoryError),系统类加载器;

9. 反映java虚拟机内部情况的JMXBean,JVMTI中注册的回调,本地代码缓存等; 

10. 除了这些固定的GC Roots集合以外,根据用户所选用的的垃圾收集器以及当前回收的内存区域不同,还可以有其他对象"临时性的"加入,共同构成完整的GC Roots集合.比如:分代收集和局部回收;

11. 如果只针对Java堆中的某一块区域进行垃圾回收(比如:典型的只针对新生代),必须考虑到内存预期是虚拟机自己的实现细节,更不是孤立封闭的,这个区域的对象完全有可能被其他区域的对象所引用,这时候就需要一并将关键的区域对象也加入GC Roots集合中去考虑,才能保证可达性分析的准确性;

* 如果要使用可达性分析算法来判断内存是否可回收,那么分析工作必须在一个能保障一致性的快照中进行;这点也是导致GC进行时必须"Stop The World"的一个重要原因;
  * 即使是号称几乎不会发生停顿的CMS收集器,枚举根节点时也是必需要停顿的;

 ### 2.2.3.对象的finalization的机制

* Java语言提供了对象终止(finalization)机制来允许开发人员提供**对象被销毁之前的自定义处理逻辑**;
* 当垃圾回收器发现没有引用指向一个对象,即:垃圾回收此对象之前,总会先调用这个对象的`finalize()`方法;
* `finalize()`方法允许在子类中被重写,用于在对象被回收时进行资源释放.通常在这个方法中进行一些资源释放和清理的工作,比如关闭文件,套接字和数据库连接等;
* 永远不要主动调用某个对象的`finalize()`方法,应该交给垃圾回收机制调用.理由如下:
  * 在`finalize()`时可能会导致对象复活;
  * `finalize()`方法的执行时间是没有保障的,它完全由GC线程决定,极端情况下,若不发生GC,则`finalize()`方法将没有执行机会;
  * 一个糟糕的`finalize()`会严重影响GC性能;
* 由于`finalize()`方法的存在,虚拟机中的对象一般处于三种可能的状态;
  * 可触及的:从根节点开始,可以到达这个对象;
  * 可复活的:对象的所有引用都被释放,但是对象有可能在`finalize()`中复活;
  * 不可触及的:对象的`finalize()`被调用,并且没有复活,那么就会进入不可触及状态.不可触及的对象不可能被复活,因为`finalize()`只会被调用一次;
* 判定一个对象objA是否可回收,至少要经历两次标记过程;
  * 如果对象objA到GC Roots没有引用链,进行第一次标记;
  * 进行筛选,判断此对象是否有必要执行`finalize()`方法;
    * 如果对象objA没有重写`finalize()`方法,或者`finalize()`方法已经被虚拟机调用过,则虚拟机视为"没有必要执行",objA被判定为不可触及的;
    * 如果对象objA重写`finalize()`方法,且还未执行过,那么objA会被插入到F-Queue队列中,由一个虚拟机自动创建,低优先级的Finalizer线程触发其`finalize()`方法执行;线程如下
      * ![image-20210825221902899](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210825221902899.png)
    * 稍后GC会对F-Queue队列中的对象进行第二次标记.如果objA在`finalize()`方法中与引用链上的任何一个对象建立了联系,那么在第二次标记时,objA会被移出"即将回收"集合,对象会再次出现没有引用存在的情况.在这个情况下,finalize方法不会被再次调用,对象会直接变成不可触及的状态;也就是说,一个对象的`finalize()`方法只会被调用一次;

### 2.2.4.通过MAT查看GC Roots

![image-20210826141326386](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826141326386.png)

## 2.3.垃圾清除阶段-标记清除算法

执行过程:当堆中的有效内存空间被耗尽的时候,就会停止整个程序(Stop the world),然后进行两项工作,第一项是标记,第二项是清除;

* 标记:Collector从引用根节点开始遍历,标记所有被引用的对象,一般是在对象的Header中记录为可达对象;
* 清除:Collector对堆内存从头到尾进行线性的遍历,如果发现某个对象在Header中没有标记为可达对象,则将其回收;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826155201232.png" alt="image-20210826155201232" style="zoom:50%;" />

缺点:

* 效率不高;
* 在进行GC的时候,需要停止整个应用程序,导致用户体验差;
* 这种方式清理出来的空闲内存是不连续的,产生内存碎片,需要维护一个空闲列表;
* 注意:这里所谓的清除并不是真的置空,而是把需要清除的对象的对象地址保存在空闲的地址列表中.下次有对象需要加载时候,判断垃圾的位置是否够,如果够,就存放;

## 2.4.垃圾清除阶段-复制算法

将活着的内存空间分为两块,每次只使用其中一块,在垃圾回收时将正在使用的内存中的存活独享复制到未被使用的内存块中,之后清除正在使用的内存块中的所有对象,交换两个内存的角色,最后完成垃圾回收;

 <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826163808400.png" alt="image-20210826163808400" style="zoom: 50%;" />

* 优点:
  * 没有标记和清除过程,实现简单,运行高效;
  * 复制过去以后保证空间的连续性,不会出现"碎片"问题;
* 缺点:
  * 此算法的缺点是很明显的,需要两倍的内存空间;
  * 对于G1这种分拆称为大量region的GC,复制而不是移动,意味着GC需要维护region之间对象引用关系,不管是内存占用或者时间开销也不小;
* 适用:
  * 如果系统中的垃圾独享很多,复制算法需要复制的存活对象数量不大;

## 2.5.垃圾清除阶段-标记压缩(整理)算法

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826175900622.png" alt="image-20210826175900622" style="zoom:50%;" />

* 第一阶段和标记清除算法一样,从根节点开始标记所有被引用对象;
* 第二阶段将所有的存活对象压缩到内存的一端,按顺序排放;之后,清理边界外所有的空间;

标记压缩是**移动式**的,可以看到标记的存活对象将会被整理,按照内存地址依次排列,而未被标记的内存会被清理掉.如此一来,当我们需要给新对象分配内存时,JVM只需要持有一个内存的起始地址即可,这个比维护一个空闲列表少了很多开销;

* 优点:
  * 消除了标记-清除算法中,内存区域分散的特点,需要给新对象分配内存时候,JVM只需要持有一个内存的起始地址即可;
  * 消除了复制算法中,内存减半的代价;
* 缺点:
  * 效率较低;
  * 移动对象的同时,如果对象被其他对象引用,还需要调整引用的地址;
  * 移动的过程中,需要全程暂停用户应用程序,STW;

## 2.6.三种算法对比

![image-20210826195349570](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826195349570.png)

## 2.7.分代收集算法

目前机会所有的GC都是采用分代收集算法执行垃圾回收;

在HotSpot中,基于分代的概念,GC所使用的内存回收算法必须结合年轻代和老年代各自的特点;

* 年轻代
  * 年轻代特点:区域相对老年代较小,对象声明周期短,存活率低,回收频繁;
  * 这种情况复制算法的回收整理速度是最快的,复制算法的效率只喝当前存活对象大小有关,因此很适合于年轻代的回收.而复制算法内存利用率不高的问题,通过hotspot中的survivor的设计得到缓解;
* 老年代
  * 老年代特点:区域较大,对象生命周期长,存活率高,回收不及年轻代频繁;
  * 这种情况存在大量存活率高的对象,复制算法明显不合适,一般都是由标记-清除或者是标记标记-整理的混合实现;
    * Mark阶段的开销与存活对象的数量成正比;
    * Sweep阶段的开销与所管理区域的大小成正比;
    * Compact阶段的开销与存活对象的数据成正比;
  * 以HotSpot中的CMS回收器为例,**CMS是基于Mark-Sweep实现的**,对于对象的回收效率很高.而对于碎片问题,CMS采用基于Mark-Compact算法的Serial Old回收期作为补偿措施:**当内存回收不好(碎片导致的Concurrent Mode Failure),将采用Serial Old执行Full GC以达到对老年代内存的整理**; 

## 2.8.增量收集算法

如果一次性将所有的垃圾进行处理,需要造成系统长时间的停顿,那么就可以让垃圾收集线程和应用程序线程交替执行.**每次,垃圾收集线程只收集一小片区域的内存空间,接着切换到应用程序线程,依次反复,直到垃圾收集完成**;

**增量收集算法的基础仍然是传统的标记-清除和复制算法,增量收集算法通过对线程间冲突的妥善处理,允许垃圾收集线程以分阶段的方式完成标记,清理和复制工作;**

缺点:使用这种方式,由于在垃圾回收过程中,间断性地还执行了应用程序代码,所以能减少系统的停顿时间.但是,因为线程切换和上下文转换的消耗,会使垃圾回收的总体成本上升,造成系统吞吐量的下降;

## 2.9.分区算法

为了更好地控制GC产生的停顿时间,将一块大的内存区域分割成多个小块,根据目标的停顿时间,每次合理地回收若干个小区间,而不是整个堆空间,从而减少一次GC所产生的停顿.每一个小区间都独立使用,独立回收.这种算法的好处是可以控制一次回收多少个小区间;  

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826225117881.png" alt="image-20210826225117881" style="zoom:50%;" />

# 3.垃圾回收的相关概念

## 3.1.`System.gc()`

* 在默认情况下,通过`System.gc()`或者`Runtime.getRuntime().gc()`的调用,会显式触发Full GC;同时对老年代和新生代进行回收,尝试释放被丢弃对象占用的内存;
* `System.gc()`调用附带一个免责声明,无法保证对垃圾收集器的调用;
* JVM实现者可以通过`System.gc()`调用来决定JVM的GC行为.而一般情况下,垃圾回收应该是自动进行的,无须手动触发,否则就太过麻烦了;

## 3.2.内存溢出

* 没有空闲内存的情况:说明Java虚拟机的堆内存不够.原因有二:
  * Java虚拟机的堆内存设置不够:可能存在内存泄漏问题,也很有可能是堆的大小不合理;
  * 代码中创建了大量大对象,并且长时间不能被垃圾收集器收集;
* 在抛出OOM之前,通常垃圾收集器会被触发,尽可能去清理出空间;
* 也不是在任何情况下垃圾收集器都会触发,当我们分配一个超大对象,超过堆的最大值,JVM判断出垃圾收集器并不能解决这个问题,直接OOM;

## 3.3.内存泄漏

**严格来说,只有对象不会被程序用到,但是GC又不能回收它们的情况才叫内存泄漏;**

**宽泛意义上说,实际情况下一些不太好的实践(或疏忽)导致对象的生命周期变得很长甚至导致OOM;**

![image-20210826235535921](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210826235535921.png)

内存泄漏举例:

1. 单例模式:单例的生命周期和应用程序是一样长的,所以单例程序中,如果持有对外部对象的引用的话,那么这个外部对象是不能被回收的,会导致内存泄漏的产生;
2. 一些提供close的资源未关闭:如数据库连接,网络连接(socket)和io连接必须手动close;

## 3.4.Stop The World

GC事件发生过程中,会产生引用程序的停顿.停顿产生时整个应用程序线程都会被暂停,没有任何响应,这个停顿称为STW;

* 可达性分析算法中枚举根节点(GC Roots)会导致所有Java执行线程停顿; 
  * 分析工作必须在一个能确保一致性的快照中进行;
  * 一致性指整个分析期间整个执行系统看起来像被冻结在某个时间点上;
  * 如果出现分析过程中对象引用关系还在不断变化,则分析结果的准确性无法保证;
* 被STW中断的应用程序线程会在完成GC之后恢复;
* STW事件和采用哪款GC无关,所有的GC都有这个事件,哪怕是G1也不能完全避免Stop-the-world情况发生,只能说垃圾回收器越来越优秀,回收效率越来越高,尽可能地缩短了暂停时间;
* STW是JVM在后台自动发起和自动完成的.在用户不可见的情况下,把用户正常的工作线程全部停掉;
* 开发中不要使用`System.gc()`,会导致Stop-the-world的发生;

## 3.5.垃圾回收的并发与并行

* 并行(Parallel):值多条垃圾收集线程并行工作,但此时用户线程仍处于等待状态;
  * 如:ParNew,Parallel Scavenge,Parallel Old;
  * 在单核CPU下并行可能更慢;
* 串行(Serial):相较于并行的概念,单线程执行;如果内存不够,则程序暂停,启动JVM垃圾回收器进行垃圾回收,回收完,再启动程序的线程;

* 并发(Concurrent):指用户线程与垃圾收集线程同时执行(但不一定是并行的,可能是交替进行),垃圾回收线程在执行不会停顿用户程序的运行;
  * 用户程序在继续运行,而垃圾收集程序线程运行于另一个CPU上;
  * 如CMS,G1;

## 3.6.安全点与安全区域

### 3.6.1.安全点

程序执行时并非在所有地方都能停顿下来开始GC,只有在特定的位置才能停顿下来开始GC,这些位置称为"安全点(safepoint)";

**safe point的选择很重要,如果太少可能导致GC等待的时间太长,如果太频繁可能导致运行时的性能问题.大部分指令的执行时间都非常短暂,通常会根据"是否具有让程序长时间执行的特征"为标准,**比如选择一些执行时间较长的指令作为Safe Point,如方法调用,循环跳转和异常跳转;

如何在GC发生时,检查所有线程都跑到最近的安全点停顿下来?

* 抢先式中断:(目前没有虚拟机采用)
  * 首先中断所有线程,如果还有线程不在安全点,就恢复线程,让线程跑到安全点;
* 主动式中断;
  * 设置一个中断标志,各个线程运行到Safe Point的时候主动轮询这个标志,如果中断标志为真,则将自己进行中断挂起;

### 3.6.2.安全区域

当程序"不执行"的时候,例如线程处于Sleep状态或者Blocked状态,这时候线程无法响应JVM的中断请求,走到安全点去中断挂起,JVM也不太可能等待线程被唤醒,这时候需要安全区域;

安全区域是指在一段代码片段中,对象的引用关系不会发生变化,在这个区域中的任何位置开始GC都是安全的.我们可以把Safe Region看作是被扩展了的Safepoint;

执行时候:

* 当线程运行到Safe Region代码时,首先标识已经进入了Safe Region,如果这段时间发生GC,JVM会忽略标识为Safe Region状态的线程;
* 当线程即将离开Safe Region到时候,会检查JVM是否已经完成GC,如果完成了,继续执行,否则线程必须等待直到收到可以安全离开Safe Region的信号为止;

## 3.7.引用关系

JDK1.2之后,Java对引用的概念进行扩充,将引用分为强引用(Strong Reference),软引用(Soft Reference),弱引用(Weak Reference),虚引用(Phantom Reference),这4中强度依次减弱;

![image-20210827103349414](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827103349414.png)

软引用,弱引用和虚引用对应上图的类.

* 强引用:最传统的应用定义,是指在程序代码中普遍存在的引用复制,类似`Object obj = new Object()`这种引用关系,无论任何情况下,只要强引用关系还存在,垃圾收集器就永远不会回收掉被引用的对象;
* 软引用:在系统将要发生内存溢出之前,将会把这些对象列如回收范围之中进行第二次回收.如果这次回收后还没有足够的内存,才会抛出内存溢出异常;
* 弱引用:被弱引用关联的对象只能生存到下一次垃圾收集之前,当垃圾收集器工作时,无论内存空间是否足够,都会回收掉被弱引用关联的对象;
* 虚引用:一个对象是否有虚引用的存在,完全不会对其生存时间构成影响,也无法通过虚引用来获得一个对象的实例,为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知;

### 3.7.1.强引用

强引用是默认的引用类型化,是可触及的,垃圾收集器永远不会回收掉被引用的对象;

相对的,软引用,弱引用和虚引用的对象是软可触及,弱可触及和虚可触及的,在一定条件下,都是可以被回收的.所以,强引用是造成Java内存泄漏的主要原因之一;

### 3.7.2.软引用

**内存不足即回收**

软引用用来描述一些还有用,但非必须的对象.只被软引用关联着的对象,在系统将要发生内存泄漏异常,会把这些对象列进回收范围之中进行第二次回收,如果这次回收还没有足够的内存,才会抛出内存溢出异常;

软引用通常用来实现内存敏感的缓存.比如:高速缓存就用到软引用.

```java
Object obj = new Object();//声明强引用;
SoftReference<Object> sf = new SoftReference<Object>(obj);
obj = null;//销毁强引用;
```

上述代码这样操作的话,obj对象只会有一个软引用,不会有强引用;

软引用和弱引用在Guava Cache中被普遍使用,如下图:

![image-20210827154932358](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827154932358.png)

如下是软引用的测试

~~~java
public class SoftReferenceTest {
    public static class User {
        public User(int id, String name) {
            this.id = id;
            this.name = name;
        }

        public int id;
        public String name;

        @Override
        public String toString() {
            return "[id=" + id + ", name=" + name + "] ";
        }
    }

    public static void main(String[] args) {
        //创建对象，建立软引用
				//SoftReference<User> userSoftRef = new SoftReference<User>(new User(1, "fechin"));
        //上面的一行代码，等价于如下的三行代码
        User u1 = new User(1,"fechin");
        SoftReference<User> userSoftRef = new SoftReference<User>(u1);
        u1 = null;//取消强引用


        //从软引用中重新获得强引用对象
        System.out.println(userSoftRef.get());

        System.gc();
        System.out.println("After GC:");
        //垃圾回收之后获得软引用中的对象
        System.out.println(userSoftRef.get());//由于堆空间内存足够，所有不会回收软引用的可达对象。

        try {
            //让系统认为内存资源紧张、不够
						//byte[] b = new byte[1024 * 1024 * 7];
            byte[] b = new byte[1024 * 7168 - 635 * 1024];
        } catch (Throwable e) {
            e.printStackTrace();
        } finally {
            //再次从软引用中获取数据
            System.out.println(userSoftRef.get());//在报OOM之前，垃圾回收器会回收软引用的可达对象。
        }
    }
}
~~~

### 3.7.3.弱引用

被弱引用关联的对象只能生存到下一次垃圾收集发生为止.在系统GC时,只要发现弱引用,不管系统堆空间使用是否充足,都会回收掉只被弱引用关联的对象.但是,由于垃圾回收器的线程通常优先级很低.因此,并不一定能很快地发现持有的弱引用对象.这种情况下,弱引用对象可以存在较长的时间.

```java
Object obj = new Object();//声明强引用;
WeakReference<Object> wr = new WeakReference<Object>(obj);
obj = null;//销毁强引用;
```

弱引用对象与软引用对象的最大不同在于,当GC在进行回收的时候,需要通过算法检查是否回收软引用对象,而对于弱引用对象,GC总是进行回收,弱引用对象更容易,更快被GC回收;

如下是WeakReference的部分源码:

![image-20210827170801372](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210827170801372.png)

### 3.7.4.虚引用

一个对象是否有虚引用的存在,完全不会决定对象的生命周期.如果一个对象仅有虚引用,那么它和没有引用几乎是一样的,随时都可能被垃圾回收.它不能单独使用,也无法通过虚引用来获取被引用的对象.当试图通过虚引用的`get()`方法获取对象时,总是null;**为一个对象设置虚引用关联的唯一目的在于跟踪垃圾回收的过程.比如:能在这个对象被收集器回收时收到一个系统通知**;

* 虚引用必须和引用队列一起使用.**虚引用在创建时必须提供一个引用队列作为参数.当垃圾回收器准备回收一个对象时,如果发现它还有虚引用.就会在回收对象后,将这个虚引用加入引用队列,以通知应用程序对象的回收情况**;
* 由于虚引用可以跟踪对象的回收时间,因此,可以将一些资源释放操作放置在虚引用中执行和记录;

~~~java
Object obj = new Object();
ReferenceQueue phantomQueue = new ReferenceQueue();
PhantomReference<Object> pf = new PhantomReference<>(obj,phantomQueue);
obj = null;
~~~

~~~java
public class PhantomReferenceTest {
    public static PhantomReferenceTest obj;//当前类对象的声明
    static ReferenceQueue<PhantomReferenceTest> phantomQueue = null;//引用队列

    public static class CheckRefQueue extends Thread {
        @Override
        public void run() {
            while (true) {
                if (phantomQueue != null) {
                    PhantomReference<PhantomReferenceTest> objt = null;
                    try {
                        objt = (PhantomReference<PhantomReferenceTest>) phantomQueue.remove();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    if (objt != null) {
                        System.out.println("追踪垃圾回收过程：PhantomReferenceTest实例被GC了");
                    }
                }
            }
        }
    }

    @Override
    protected void finalize() throws Throwable { //finalize()方法只能被调用一次！
        super.finalize();
        System.out.println("调用当前类的finalize()方法");
        obj = this;
    }

    public static void main(String[] args) {
        Thread t = new CheckRefQueue();
        t.setDaemon(true);//设置为守护线程：当程序中没有非守护线程时，守护线程也就执行结束。
        t.start();

        phantomQueue = new ReferenceQueue<PhantomReferenceTest>();
        obj = new PhantomReferenceTest();
        //构造了 PhantomReferenceTest 对象的虚引用，并指定了引用队列
        PhantomReference<PhantomReferenceTest> phantomRef = new PhantomReference<PhantomReferenceTest>(obj, phantomQueue);

        try {
            //不可获取虚引用中的对象
            System.out.println(phantomRef.get());

            //将强引用去除
            obj = null;
            //第一次进行GC,由于对象可复活，GC无法回收该对象
            System.gc();
            Thread.sleep(1000);
            if (obj == null) {
                System.out.println("obj 是 null");
            } else {
                System.out.println("obj 可用");
            }
            System.out.println("第 2 次 gc");
            obj = null;
            System.gc(); //一旦将obj对象回收，就会将此虚引用存放到引用队列中。
            Thread.sleep(1000);
            if (obj == null) {
                System.out.println("obj 是 null");
            } else {
                System.out.println("obj 可用");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
~~~

### 3.7.5.终结器引用

* 它用以实现对象的`finalize()`方法,也可以成为终结器引用;
* 在GC时,终结器引用入队.由Finalizer线程通过终结器引用找到被引用对象并调用它的`finalize()`方法,第二次GC才能回收被引用对象;

