# JVM详解03

# 1.对象的实例化

* 创建对象的方式:
  * `new `;
  * `Class.newInstance()`;反射的方式,只能调用public和空参的构造器;
  * `Constructor.newInstance(xxx)`;反射的方式,可以调用空参,带参,权限没有要求的构造器;
  * `clone()`:当前类需要实现Cloneable接口;
  * 反序列化;
* 创建对象的步骤
  1. 判断对象对应的类是否加载,链接,初始化
     * 当虚拟机遇到一条new指令,首先去检查这个指令的参数能否再Metaspace的常量池中定位到一个类的符号引用,并且检查这个符号引用代表的类是否已经被加载,解析,初始化(即判断类元信息是否存在);如果没有,那么在双亲委派模式下,使用当前类加载器以ClassLoader+包名+类名为key进行查找对应的.class文件,如果没有找到文件,则抛出ClassNotFoundException异常,如果找到,则进行类加载,并生成对应的Class类对象;
  2. 为对象分配内存;
     1. 如果内存规整;指针碰撞;
        * 如果内存是规整的,那么虚拟机采用的是指针碰撞法来为对象分配内存,意思是所有用过的内存在一边,空间的内存在另外一边,中间放着一个指针作为分界点的指示器,分配内存仅仅是吧指针向空闲那边移动一段与对象大小相等的距离而已;如果垃圾收集器选择的是Serial,ParNew这种压缩算法的,虚拟机采用这种分配方式;
     2. 如果内存不规整,虚拟机需要维护一个列表进行空闲列表分配;
  3. 处理并发安全问题;
     1. 采用CAS失败重试,区域加锁保证更新的原子性;
     2. 每个线程预先分配一块TLAB,`-XX:+/-UseTLAB`参数来设定;
  4. 初始化分配到的空间:所有属性设置默认值, 保证对象实例字段在不赋值时候可以直接使用;
  5. 设置对象的对象头;
     * 将对象的所属类(类的元数据信息),对象的HashCode和对象的GC信息,锁信息等数据存储在对象的对象头中;
  6. 执行\<init\>方法进行初始化(这里的init包括显示初始化,代码块中的初始化,构造器初始化);

# 2.对象的内存布局 

![image-20210819134934493](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819134934493.png)

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819135512875.png)

# 3.对象的访问定位

JVM是通过栈上的reference访问到其内部的对象实例;

访问方式主要有两种:

* 句柄访问;

  * 好处:  reference中存储稳定句柄地址,对象被移动(垃圾收集时移动对象很普遍)时只会改变句柄中实例数据指针即可,reference本身不需要被修改;

  ![image-20210819141429179](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819141429179.png)

* 指针访问;(HotSpot采用);

  ![image-20210819141450502](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819141450502.png)

# 4.直接内存

`java process memory = java heap+native memory(metaspace+direct memory)`;

* 直接内存是在Java堆外的,直接向系统申请的内存空间;
* 来源于NIO,通过存在堆中的DirectByteBuffer操作Native内存;
* 访问直接内存的速度会优与Java堆,即读写性能高;
  * 处于对性能的考虑,读写频繁的场合可能会考虑使用直接内存;
  * Java的NIO库允许Java程序使用直接内存,用于数据缓冲区;
* 缺点:
  * 分配回收成本高;
  * 不受JVM内存回收管理;
* 直接内存大小可以通过MaxDirectMemorySize设置;如果不指定,默认与堆的最大值`-Xmx`参数值一致;

## 4.1.非直接缓冲区

读写文件,需要与磁盘交互,需要由用户态切换到内核态,如下图,需要两份内存存储重复数据,效率低;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819143223799.png" alt="image-20210819143223799" style="zoom:67%;" />

## 4.2.直接缓冲区

使用NIO时,操作系统划出的直接缓存区可以被Java代码直接访问,NIO适合对大文件的读写操作;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819143317841.png" alt="image-20210819143317841" style="zoom:67%;" /> 

# 5.执行引擎

虚拟机是一个相对物理机的概念,这两种机器都有代码执行能力,其区别是物理机的执行引擎是直接建立在处理器,缓存,指令集和操作系统层面上的,而虚拟机的执行引擎是由软件自行实现的,因此可以不受物理条件制约地定制指令集与执行引擎的结构体系;能够执行那些不被硬件直接支持的指令集格式;

JVM的主要任务时负责装载字节码到其内部,但字节码并不能直接运行在操作系统之上,因为字节码指令并非等价于本地机器指令.执行引擎的任务就是将字节码指令解释/编译为对应平台上的本地机器指令; 

## 5.1.Java代码编译和执行的过程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819151824691.png" alt="image-20210819151824691" style="zoom:50%;" />

橙色部分是Javac将java编译成.class文件的过程;绿色是解释执行,蓝色是编译执行;

## 5.2.机器码, 指令,汇编语言

![image-20210819153354667](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819153354667.png)

* 机器码:用二进制编码的方式表示的指令,叫做机器指令码;
* 指令:由于机器码是由0和1组成的二进制序列,可读性太差,于是有了指令(一般为英文简写,如mov,inc等);
  * 指令集:不同的硬件平台,各自支持的指令,是有差别的;因此每个平台所支持的指令,称为对应平台的指令集;
* 汇编语言:用助记符代替机器指令的操作码,用地址符号或标号代替指令或操作数的地址;在不同的硬件平台,汇编语言对应着不同的机器语言指令集,通过汇编过程转换成机器指令;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819154803293.png" alt="image-20210819154803293" style="zoom: 33%;" />

* 字节码:是一种中间状态(中间码)的二进制代码;需要直译器转议后才能成为机器码;

## 5.3.解释器

Java的发展历史中,一共有两套解释执行器,古老的**字节码解释器**和普遍使用的**模板解释器**;

* 字节码解释器在执行时通过纯软件代码模拟字节码的执行,效率低下;
* 模板解释器将每一条字节码和一个模板函数相关联,模板函数中直接产生了这条字节码执行时的机器码,从而很大程度上提高了解释器的性能.在HotSpot中,解释器主要由Interpreter模块和Code模块构成;
  * Interpreter模块:实现了解释器的核心功能;
  * Code模块:用于管理HotSpot VM在运行时生成的本地机器指令;

解释器的优点:

* 当程 序启动后,解释器可以马上发挥作用,省去编译时间,立即执行;编译器想要发挥作用,需要把代码编译成本地代码,需要时间;

## 5.4.JIT编译器

当编译器启动的时候,解释器可以首先发挥作用,而不必等待即使编译器全部编译完成再执行;随着程序运行时间的推移,即使编译器逐渐发挥作用,根据热点探测功能,将有价值的字节码编译为本地机器指令,换取更高的程序执行效率;

在Java中编译器的概念有三种:

* 前端编译器:把java文件转变为.class文件.如Javac;
* JIT编译器:把字节码转变为机器码;如HotSpot的C1,C2编译器;
* AOT编译器:(Ahead Of Time Compiler)静态前提编译器,直接把.java文件编译成本地机器代码的过程;如GNU Compiler for the Java(GCJ);

是否需要启动JIT编译器将字节码直接编译为对应平台的本地机器指令,需要根据代码被调用执行频率确定.JIT编译器在运行时会针对频繁被调用的**热点代码**做出**深度优化**,将其直接编译为对应平台的本地机器指令,以提升性能;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210819170910362.png" alt="image-20210819170910362" style="zoom:50%;" />计数器用于统计方法被调用的次数`-XX:CompileThreshold`来人为设定;

如果不做任何设置,方法调用计数器统计的并不是方法被调用的绝对次数,当超过一定的时间限度,如果方法的调用次数仍然不足以让它提交给即时编译器编译,那这个方法的调用计数器就会被减少一半,这个过程称为调用计数器热度的**衰减**,而这段时间就称为此方法的**半衰周期**;

`-XX:-UseCounterDecay`关闭热度衰减;`-XX:CounterHalfLifeTime`参数设置半衰周期事件,单位为秒;

### 5.4.1.C1和C2编译器

HotSpot有两个JIT编译器,分贝位Client Compiler和Server Compiler;

* client Compiler即在Client模式下,C1编译器;
  * **C1编译器**会对字节码进行简单和可靠的优化,耗时短;
* server Compiler:在Server模式下,C2编译器;
  * **C2编译器**进行耗时较长的优化,以及激进的优化;

> C1和C2编译器不同的优化策略:
>
> * C1:
>   * 方法内联:将引用的函数代码编译到引用点出,减少栈帧的生成,减少参数传递以及跳转; 
>   * 去虚拟化:对唯一的实现类进行内联;
>   * 冗余消除:运行期间把一些不会执行的代码折叠掉;
> * C2:C2的优化主要在全局层面,逃逸分析是优化的基础;
>   * 标量替换:用标量值代替聚合对象的属性值;
>   * 栈上分配;对于未逃逸的对象分配在栈;
>   * 同步消除;清除同步操作,通常指的是synchronized;

分层编译策略:JDK7之后,默认开启分层编译策略,由C1和C2共同执行;

## 5.5.HotSpot设置程序执行方式

默认情况下HotSpot VM采用的是解释器和即时编译器并存的架构;

* `-Xint`:完全采用解释器模式执行程序;
* `-Xcomp`:完全采用即时编译器执行,如果即时编译出现问题,解释器会介入;
* `-Xmixed`:混合模式;

## 5.6.Graal编译器和AOT编译器

JDK10起,HotSpot加入了一个即时编译器:Graal编译器;(与C1,C2并列的概念)

JDK9引入了AOT编译器(与JIT并列的概念),jaotc,借助Graal编译器,将Java类文件转化为机器码,并存放在动态共享库中;

# 6.StringTable

* String的String Pool是一个固定大小的HashTable,默认值大小长度是10009.如果放入String Pool的String非常多,就会造成Hash冲突严重,从而导致链表会很长,而链表长了之后直接会造成的影响就是当调用`String.intern()`时候回性能大幅下降;
* 使用`-XX:StringTableSize`可设置StringTable的长度;
* 在JDK6中StringTable是固定的,1009的长度,所以如果常量池中的字符串过多就会导致效率下降;
* 在JDK7中,StringTable的长度默认值是60013,JDK8开始,1009是可设置的最小值;

## 6.1.String内存分配

8种基本数据类型和String,这些类型为了它们在运行过程中速度更快,更节省内存,都提供了常量池;

常量池就类似一个Java系统级别的缓存,8种基本数据类型的常量池都是系统协调的.String类型的常量池比较特殊,使用方法有两种:

* 直接使用双引号声明的String对象会直接存储在常量池中`String info = "fechin"`;
* 如果不是用双引号声明的String对象,可以使用String提供的`intern()`方法;

![image-20210820111526011](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210820111526011.png)

可以在IDEA中进行debug中观察内存;

## 6.2.String拼接

* 常量与常量的拼接结果在常量池,原理是编译期优化;
* 常量池中不会存在相同内容的常量;
* 只要其中有一个是变量,结果就在堆中.变量拼接的原理是StringBuilder;
* 如果拼接的结果调用`intern()`方法,则主动将常量池中还没有的字符串对象放入池中,并返回此对象地址;

如下是前端编译器优化String拼接的代码:

~~~java
public void test1(){
        String s1 = "a" + "b" + "c";//编译期优化：等同于"abc"
        String s2 = "abc"; //"abc"一定是放在字符串常量池中，将此地址赋给s2
        /*
         * 最终.java编译成.class,再执行.class
         * String s1 = "abc";
         * String s2 = "abc"
         */
        System.out.println(s1 == s2); //true
        System.out.println(s1.equals(s2)); //true
    }
~~~

~~~java
public void test2(){
        String s1 = "javaEE";
        String s2 = "hadoop";

        String s3 = "javaEEhadoop";
        String s4 = "javaEE" + "hadoop";//编译期优化
        //如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
        String s5 = s1 + "hadoop";
        String s6 = "javaEE" + s2;
        String s7 = s1 + s2;

        System.out.println(s3 == s4);//true
        System.out.println(s3 == s5);//false
        System.out.println(s3 == s6);//false
        System.out.println(s3 == s7);//false
        System.out.println(s5 == s6);//false
        System.out.println(s5 == s7);//false
        System.out.println(s6 == s7);//false
        //intern():判断字符串常量池中是否存在javaEEhadoop值，如果存在，则返回常量池中javaEEhadoop的地址；
        //如果字符串常量池中不存在javaEEhadoop，则在常量池中加载一份javaEEhadoop，并返回此对象的地址。
        String s8 = s6.intern();
        System.out.println(s3 == s8);//true
    }
~~~

 ~~~java
     public void test3(){
         String s1 = "a";
         String s2 = "b";
         String s3 = "ab";
         /*
         如下的s1 + s2 的执行细节：(变量s是临时定义的）
         ① StringBuilder s = new StringBuilder();
         ② s.append("a")
         ③ s.append("b")
         ④ s.toString()  --> 约等于 new String("ab")
 
         补充：在jdk5.0之后使用的是StringBuilder,在jdk5.0之前使用的是StringBuffer
          */
         String s4 = s1 + s2;//
         System.out.println(s3 == s4);//false
     }
 ~~~

~~~java
    /*
    1. 字符串拼接操作不一定使用的是StringBuilder!
       如果拼接符号左右两边都是字符串常量或常量引用，则仍然使用编译期优化，即非StringBuilder的方式。
    2. 针对于final修饰类、方法、基本数据类型、引用数据类型的量的结构时，能使用上final的时候建议使用上。
     */
    @Test
    public void test4(){
        final String s1 = "a";
        final String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2;
        System.out.println(s3 == s4);//true
    }
~~~

~~~java
		/*
    体会执行效率：通过StringBuilder的append()的方式添加字符串的效率要远高于使用String的字符串拼接方式！
    详情：① StringBuilder的append()的方式：自始至终中只创建过一个StringBuilder的对象
          使用String的字符串拼接方式：创建过多个StringBuilder和String的对象
         ② 使用String的字符串拼接方式：内存中由于创建了较多的StringBuilder和String的对象，内存占用更大；如果进行GC，需要花费额外的时间。

     改进的空间：在实际开发中，如果基本确定要前前后后添加的字符串长度不高于某个限定值highLevel的情况下,建议使用构造器实例化：
     StringBuilder s = new StringBuilder(highLevel);//new char[highLevel]
     */
    @Test
    public void test6(){

        long start = System.currentTimeMillis();

        //method1(100000);//4014
        method2(100000);//7

        long end = System.currentTimeMillis();

        System.out.println("花费的时间为：" + (end - start));
    }

    public void method1(int highLevel){
        String src = "";
        for(int i = 0;i < highLevel;i++){
            src = src + "a";//每次循环都会创建一个StringBuilder、String
        }
        //System.out.println(src);

    }

    public void method2(int highLevel){
        //只需要创建一个StringBuilder
        StringBuilder src = new StringBuilder();
        for (int i = 0; i < highLevel; i++) {
            src.append("a");
        }
        //System.out.println(src);
    }
~~~

~~~java
/**
 * 题目：
 * new String("ab")会创建几个对象？看字节码，就知道是两个。
 *     一个对象是：new关键字在堆空间创建的
 *     另一个对象是：字符串常量池中的对象"ab"。 字节码指令：ldc
 *
 * 思考：
 * new String("a") + new String("b")呢？
 *  对象1：new StringBuilder()
 *  对象2： new String("a")
 *  对象3： 常量池中的"a"
 *  对象4： new String("b")
 *  对象5： 常量池中的"b"
 *
 *  深入剖析： StringBuilder的toString():
 *      对象6 ：new String("ab")
 *       强调一下，toString()的调用，在字符串常量池中，没有生成"ab"
 *
 */
public class StringNewTest {
    public static void main(String[] args) {
//        String str = new String("ab");

        String str = new String("a") + new String("b");
    }
}
~~~

## 6.3.`intern()`

~~~java
public class StringIntern {
    public static void main(String[] args) {

        String s = new String("1");
        s.intern();//调用此方法之前，字符串常量池中已经存在了"1"
        String s2 = "1";
        System.out.println(s == s2);//jdk6：false   jdk7/8：false


        String s3 = new String("1") + new String("1");//s3变量记录的地址为：new String("11")
        //执行完上一行代码以后，字符串常量池中，是否存在"11"呢？答案：不存在！！
        s3.intern();//在字符串常量池中生成"11"。如何理解：jdk6:创建了一个新的对象"11",也就有新的地址。
                                            //jdk7:此时常量中并没有创建"11",而是创建一个指向堆空间中new String("11")的地址
        String s4 = "11";//s4变量记录的地址：使用的是上一行代码代码执行时，在常量池中生成的"11"的地址
        System.out.println(s3 == s4);//jdk6：false  jdk7/8：true
    }
}
~~~

关于`intern()`:

* 在JDK1.6中,将这个字符串对象尝试放入串池;
  * 如果串池中有,则不会放入.返回已有的串池中的对象的地址;
  * 如果没有,会把此对象复制一份,放入串池,并返回串池中的对象地址;
* JDK1.7起,将这个字符串对象尝试放入串池;
  * 如果串池中有,则不会放入,返回已有的串池中的对象的地址;
  * 如果没有,则会把对象的引用地址复制一份,放入串池,并返回串池中的应用地址;

  ## 6.4.StringTable的垃圾回收

 同堆的垃圾回收

## 6.5.G1的String去重操作

`UseStringDeduplication`:开启去重String,默认是不开启的;

`PrintStringDeduplicationStatistics`:打印详细的去重统计信息;

`StringDeduplicationAgeThreshold`:达到这个年龄的String对象被认为是去重的

