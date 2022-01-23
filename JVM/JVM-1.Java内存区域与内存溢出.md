# JVM-1.Java内存区域与内存溢出

# 1. 运行时数据区

![image-20210426165826528](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210426165826528.png)

## 1.1.程序计数器

可以看成是当前线程执行的字节码的行号指示器,字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令,它是程序控制流的指示器,分支,循环,跳转,异常处理,线程恢复等基础功能都需要依赖这个计数器完成;

## 1.2.虚拟机栈

虚拟机栈描述的是Java方法执行的线程内存模型:每个方法被执行的时候,Java虚拟机都会同步创建一个栈帧(Stack Frame)用于存储局部变量表,操作数栈,动态连接,方法出口等信息;每一个方法被调用直至执行完毕的过程,就对应着栈帧在虚拟机中从入栈到出栈的过程;

### 1.2.1.局部变量表

局部变量表中存放了编译期可知的各种Java虚拟机基本数据类型(boolean,byte,char,short,int,float,long,double),对象引用和returnAddress类型;

## 1.3.本地方法栈

与虚拟机栈类似,区别是虚拟机栈为虚拟机执行Java方法服务,本地方法栈为虚拟机使用到的本地(Native)方法使用;

## 1.4.堆

存放对象实例,Java堆是垃圾收集器管理的内存区域;

## 1.5.方法区

用于存放被虚拟机加载的类型信息,常量,静态变量即时编译器编译后代码缓存等数据;

在JDK 6的时候HotSpot开发团队就有放弃永久代,逐步改为采用本地内存来实现方法去的计划,到了JDK 7 的时HotSpot,已经把原本放在永久代的字符串常量池,静态变量等出.到了JDK 8的时候,完全放弃永久代的概念,改用在本地内存中实现的元空间来替代;

### 1.5.1.运行时常量池

常量池表是用于存放编译期生成的各个字面量与符号引用,这部分内容在类加载后存放到方法区的运行时常量池中.

# 2.HotSpot虚拟机的对象

## 2.1.对象的创建

1. 当Java虚拟机遇到一条字节码new指令时,首先将去检查这个指令的参数是否能在常量池定位到一个类的符号引用,并且检查这个符号引用代表的类是否被加载,解析和初始化过,如果没有,那必须先执行相应的类加载过程.

在类加载检查通过后,接卸来虚拟机将为新生对象分配内存,对象所需内存的大小在类加载完成后便可完全确定,为对象分配空间的任务实际上便等同于把一块确定大小的内存块从Java堆中画出来.

* 假设Java堆中内存是绝对规整的,所有被使用过的内存都要被放在一边,空闲的内存被放在另一边,中间放着一个指针作为分界点的指示器,那所分配的内存就仅仅把指针向空闲空间方向挪动一段与对象大小相等的距离,这种是**指针碰撞**;
* 假设Java堆中的内存并不是规整的,已被使用的内存和空闲内存相互交错在一起,虚拟机就必须维护一个列表,记录上哪些内存块是可用的,在分配的时候从列表中找到一块足够大的空间划分给对象实例,并更新列表记录,这种是**空闲列表**;
* *选择哪种方式由Java堆是否规整决定,而Java堆是否规整由所采用的垃圾收集器是否带有空间压缩整理的能力所决定*;

2. 内存分配完成后,虚拟机必须要将分配到的内存空间(不包括对象头)都初始化为零值,如果使用了TLAB的话,这一项工作也可以提前到TLAB分配时顺便进行.这步操作保证了对象的实例字段在Java代码中可以不赋初始值就可以直接使用.

3. 接下来,Java虚拟机对对象进行必要的设置,例如这个对象是哪个类的实例,如何才能找到类的元数据信息,都放在对象头中;
4. 上面的过程全部完成后,开始执行构造函数`<init>()`;

### 2.1.1.对象创建的线程安全问题

方案:

1. 对分配内存空间的动作进行同步处理--实际上虚拟机是采用CAS配上失败重试的方案来保证更新操作的原子性;
2. 把内存分配的动作按照线程划分在不同空间中进行,即每个线程在Java堆中预先分配一小块内存,称为本地线程分配缓冲(Thread Local Allocation Buffer,TLAB),哪个线程要分配内存,就在哪个线程的本地缓冲区分分配,只有本地缓冲区用完了,分配新的内存才需要同步锁定.可以通过`-XX:+/-UseTLAB`参数来设定;

## 2.2.对象的内存布局

HotSpot虚拟机,对象在堆内存中的布局可以划分为:对象头(Header),实例数据(Instance Data)和对齐填充(Padding);

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210427171649.png)

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210427171732.png)

## 2.3.对象的访问的定位

主流的访问方式分为:使用句柄和直接指针;

1. **句柄**:Java堆将可能划分出一块内存来作为句柄池,reference中存储就是对象的访问句柄地址,而句柄中包含了对象实例数据与类型数据的具体的地址信息;
2. **直接指针**:对象实例数据存放对象类型数据的指针;

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210427173025.png)

优劣势:

1. 使用句柄好处就是reference中存储的是稳定句柄的地址,在对象移动时只会改变句柄中的实例数据指针,reference不需要修改;
2. 使用直接指针来访问的好处就是速度更快,节省了一次指针定位的时间开销;

HotSpot主要进行直接指针进行访问;

# 3.OutOfMemoryError异常

## 3.1.堆溢出

~~~java
public class HeapOOM {

    static class OOMObject{

    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();

        while (true){
            list.add(new OOMObject());
        }
    }
}
~~~

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210428101505.png)

运行时加入参数`-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError`;我们可以使用Eclipse Memory Analyzer或者idea直接打开dump下来的`.hporf`文件.

第一步首先确认内存中导致OOM的对象是否是是必要的,也就是要先分析清楚到底是内存泄漏(Memory Leak)还是内存溢出(Memory Overflow);

* 内存泄漏:是指程序在申请内存后,无法释放已申请的内存空间,一次内存泄漏似乎不会有大的影响,但内存泄漏堆积后的后果就是内存溢出;
* 内存溢出:指程序申请内存时,没有足够的内存供申请者使用;

下图是MAT的overview

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210428140108.png)

## 3.2.虚拟机栈和本地方法栈溢出

由于Hotspot并不区分虚拟机栈和本地方法栈,因此对于HotSpot来说,`-Xoss`参数来设置本地方法栈大小虽然存在,但实际没有任何效果,栈容量只能由`-Xss`参数来决定.对于虚拟机栈和本地方法栈有两种异常:

1. 如果线程请求的栈的深度大于虚拟机所允许的最大深度,将抛出StackOverflowError异常;
2. 如果虚拟机的栈内存允许动态扩展,当扩展栈容量无法申请到足够的内存时,将抛出OutOfMemoryError异常;

Java虚拟机规范明确允许Java虚拟机实现自行选择是否支持栈的动态扩展,而HotSpot虚拟机的选择是不支持动态扩展,所以在创建线程申请内存时就因无法获取足够内存而出现OutOfMemoryError异常,否则在线程运行时是不会因为扩展而导致内存溢出的,只会因为栈容量无法容纳新的栈帧而导致StackOverflowError;

~~~java
public class StackOver {
    private int stackLenght = 1;

    public void stackLeak(){
        stackLenght++;
        stackLeak();
    }

    public static void main(String[] args) {
        StackOver stackOver = new StackOver();
        try {
            stackOver.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" +stackOver.stackLenght);
            throw e;
        }
    }
}
~~~

运行时加上参数`-Xss160k`

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/20210428143427.png)

另一个实验参考`<深入理解java虚拟机>第三版`58页;

实验结果表明,无论是栈帧太大还是虚拟机栈容量太小,当新的栈帧内容无法分配的时候,HotSpot虚拟机抛出的都是StackOverflowError异常;可是如果在允许动态扩展容量大小的虚拟机上,相同的代码则会导致不一样的情况.

如果测试时不限于单线程,用过不断建立线程的方式,在HotSpot上也是可以产生内存溢出异常的,但是这样产生的内存溢出异常和栈空间是否足够并不存在任何直接的关系,主要取决于操作系统本身的内存使用情况.在这种情况下,给每个线程的栈分配的内存越大,反而越容易产生内存溢出异常.原因是:操作系统分配给每个进程的内存是有限制的,为每个线程分配到的栈内存越大,可以建立的线程数自然就越少,建立线程时就越容易把剩下的内存耗尽;

## 3.3.方法区和运行时常量池溢出

`String::intern()`是一个本地方法,它的作用是如果字符串常量池中已经包含了一个等于此String对象的字符串,则返回代表这个字符串的String对象的引用;否则,会将此String对象包含的字符串添加到常量池中,并且返回此String对象的应用.在JDK6或更早的HotSpot虚拟机中,常量池都是分配在永久代中,我们可以通过`-XX:PermSize`和`-XX:MaxPermSize`限制永久代的大小,即可限制其中常量池的容量;

从JDK7开始,原本存放在永久代的字符串常量池被移至Java堆中,所以在JDK7及以上版本,限制方法区的容量毫无意义,使用`-Xmx`参数限制就可以看到`OutOfMemoryError:Java heap space`;

在JDK8开始,元空间取代了永久代,在默认情况下很难出现异常,HotSpot还提供了一些参数作为元空间的防御措施;

1. `-XX:MaxMetaspaceSize`:设置元空间最大值,默认是-1,不受限制,或者说只会手限制于本地内存大小;
2. `-XX:MetaspaceSize`:指定元空间的初始空间大小,以字节为单位,达到该值就会触发垃圾收集进行类型卸载,同时收集器会对该值进行调整:如果释放了大量的空间,就适当降低该值;如果释放了很少的空间,那么在不超过`-XX:MaxMetaspaceSize`的情况下,适当提高值;
3. `-XX:MinMetaspaceFreeRatio`:作用是在垃圾收集之后控制最小的元空间剩余容量的百分比,可减少因为元空间不足导致的垃圾收集频率;
4. `-XX:MaxMetaspaceFreeRatio`:用于控制最大的元空间剩余容量的百分比;

## 3.4.本机直接内存

直接内存(Direct Memory)的容量大小可通过`-XX:MaxDirectMemorySize`参数来指定,如果不指定,则默认与Java堆最大值(-Xmx指定)一致,如果发现内存溢出之后产生的Dump文件很小,而程序中又直接间接的使用DirectMemory(如NIO),那么需要考虑这方面原因.

