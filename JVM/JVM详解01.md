#  JVM详解01

# 1.JVM的架构

## 1.1.JVM的整体架构

![image-20210731182028995](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210731182028995.png)

* Execution Engine:执行引擎中Interpreter解释执行器和JIT编译执行器是同时存在的;
  * Interpreter:解释执行器的话解释的是.class字节码文件并执行;
  * JIT Compiler:编译执行器的话是将.class字节码编译成机器码并执行,执行的是一些热点代码,并做缓存,放在方法区中;
* PC Registers:Programme Count Register;
* Stack Area:
  * LV:local variable:局部变量表;
  * OS:operand stack:操作数栈;
  * DL:dynamic link:动态链接;
  * RA:return address:方法返回地址;
* Native Method Stack:用于本地的方法调用

## 1.2.JVM的指令集

Java编译器输入的指令流基本上是一种基于**栈的指令集架构**,另外一种指令集架构是基于寄存器的指令集架构;

* 基于栈式架构的特点:
  * 设计和实现更简单,适用于资源受限的系统;
  * 避开了寄存器的分配难题:使用零地址指令方式分配;
  * 指令流中的指令大部分是零地址指令,其执行过程依赖于操作栈.指令集更小.编译器更容易实现;
  * 不需要硬件支持,可移植性更好,更好实现跨平台;
* 基于寄存器架构的特点:
  * 典型的应用是x86二进制指令集;
  * 指令集架构完全依赖硬件,可移植性差;
  * 性能更优秀,执行更搞笑;
  * 执行时花费的指令更少;

基于栈式架构架构的指令示例:

~~~java
public class StackStructDemo {

    public static void main(String[] args) {
        int i = 2 + 3;
    }
}
~~~

上面是java代码,通过`javap -v StackStructDemo`注意这里的StackStructDemo是target中的class文件,下面就是字节码指令

~~~
Classfile /Users/fechinchu/CodeRepository/BackendDemo/jvm-demo/target/classes/StackStructDemo.class
  Last modified 2021-7-31; size 412 bytes
  MD5 checksum d4a7cca97c8aae48e55d0ba438b9fea7
  Compiled from "StackStructDemo.java"
public class StackStructDemo
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#19         // java/lang/Object."<init>":()V
   #2 = Class              #20            // StackStructDemo
   #3 = Class              #21            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               LStackStructDemo;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               args
  #14 = Utf8               [Ljava/lang/String;
  #15 = Utf8               i
  #16 = Utf8               I
  #17 = Utf8               SourceFile
  #18 = Utf8               StackStructDemo.java
  #19 = NameAndType        #4:#5          // "<init>":()V
  #20 = Utf8               StackStructDemo
  #21 = Utf8               java/lang/Object
{
  public StackStructDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LStackStructDemo;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: iconst_5
         1: istore_1
         2: return
      LineNumberTable:
        line 10: 0
        line 11: 2
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       3     0  args   [Ljava/lang/String;
            2       1     1     i   I
}
SourceFile: "StackStructDemo.java"
~~~

## 1.3.JVM的生命周期

* 虚拟机的启动
  * Java虚拟机的启动时通过引导类加载器(BootStrap ClassLoader)创建一个初始类(initial class)来完成的,这个类是由虚拟机的具体实现指定的.这个类具体由各个虚拟机来实现;
* 虚拟机的执行
  * 一个运行中的Java虚拟机有着一个清晰的任务:执行Java程序;
  * 程序开始执行时开始运行,程序结束时就停止;
  * 执行一个所谓的Java的程序的时候,真正执行的是一个叫做Java虚拟机的进程;
* 虚拟机的退出
  * 程序正常执行结束;
  * 程序在执行过程遇到了异常或错处而终止;
  * 由于操作系统出现错误而导致Java虚拟机进程终止;
  * 某线程调用Runtime类或者System类的exit方法,或者Runtime类的halt方法,并且Java安全管理器也运行这次exit或halt操作;
  * 除此之外,JNI(Java Native Interface)规范描述了JNI Invocation API来加载或者卸载Java虚拟机时,Java虚拟机的退出情况;

 # 2.JVM的常见种类

## 2.1.Sun Classic VM

* 1996年Java1.0版本,Sun公司发布Sun Classic VM的虚拟机,JDK1.4的时候被完全淘汰;
* 只提供解释器,如果使用JIT编译器,就需要进行外挂;但是一旦使用了JIT之后,就不能使用解释器;
* HotSpot内置了此虚拟机;

 ## 2.2.Exact VM

* JDK1.2,Sun公司发布Exact VM
* 虚拟机可以知道内存中某个位置的数据具体是什么类型;
* 热点探测和编译器与解释器混和工作模式;

## 2.3.HotSpot VM

* Sun/Oracle JDK和OpenJDK的默认虚拟机
* 名称中的HotSpot指的是它的热点代码探测技术
  * 通过计数器找到最具编译价值的代码,触发即使编译或栈上替换;
  * 通过编译器与解释器协同工作,在最优化的程序响应时间与最佳性能中取得平衡;

## 2.4.JRockit VM

* 专注于服务器端
  * 不关注程序启动速度,因此JRockit内部不包含解释器实现,全部代码都考即使编译器编译后实现;
* JRockit JVM是目前最快的JVM
* MissionControl服务套件,一组监控,管理和分析生产环境中的应用程序的工具;
* JDK8.0,Oracle在HotSpot的基础上,移植JRockit的部分特性;

## 2.5.J9 VM

* IBM Technology for Java VIrtual Machine,简称IT4J,内部代号:J9;

# 3.JVM类加载子系统

## 3.0.安装插件

1. jclasslib Bytecode Viewer:用来查看二进制的.class文件

2. BinEd - Binary/Hexadecimal Editor

## 3.1.类加载子系统的作用

![image-20210731224240200](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210731224240200.png)

* 类加载器子系统负责从文件系统或者网络中加载class文件,class文件在文件开头有特定的文件标识;
* ClassLoader只负责class文件的加载,至于是否可以运行,则有Execution Engine决定;
* 加载的类信息存放于一块称为方法区的内存空间.除了类的信息外,方法区中还会存放运行时常量池信息,可能还会包含字符串字面量和数字常量(这部分常量信息是class文件中常量池部分的内存映射).如下图,下图就是class文件中的常量池的信息,会映射到内存中,就叫做运行时常量池;

![image-20210731225506128](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210731225506128.png)

## 3.2.类加载器ClassLoader的角色

1. class file存放在本地硬盘上,可以理解为模板,这个模板在执行的时候是要加载到JVM中来,根据这个文件实例化出N个一模一样的实例;
2. class file加载到JVM中,被称为DNA元数据模板,放在方法区;
3. 在.class文件-->JVM-->最终称为元数据模板,此过程需要一个运输工具,Class Loader扮演一个快递员的角色;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210731230259570.png" style="zoom:50%;" />

## 3.3.类加载过程

![image-20210731230812318](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210731230812318.png)

### 3.3.1.加载过程一:Loading

1. 通过一个类的全限定名获取定义此类的二进制字节流;
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构;
3. 在内存中生成一个代表这个类的`java.lang.Class`对象,作为方法区这个类的各种数据的访问入口;

### 3.3.2.加载过程二:Linking

1. 验证(Verify)
   1. 目的在于确保Class文件的字节流所包含信息符合当前虚拟机要求,保证被加载类的正确性;
   2. 主要包括四种验证:文件格式验证,元数据验证,字节码验证,符号引用验证;
2. 准备(Prepare)
   1. 为类变量分配内存并且设置该类变量的默认初始值,即零值;
   2. 这里不包含用final修饰的static,这个就会因为final在编译的时候就会分配了,准备阶段会显式初始化;
   3. 这里不会为实例变量分配初始化,类变量会分配在方法区中,而实例变量是会随着对象一起分配到Java堆中;
3. 解析(Resolve)
   1. 将常量池内的符号引用转换为直接引用的过程;
   2. 事实上,解析操作往往会伴随着JVM在执行完初始化之后再执行;
   3. 符号引用就是一组符号来描述所引用的目标.符号引用的字面量形式明确定义在\<Java虚拟机规范\>的Class文件格式中.直接引用就是直接指向目标的指针,相对偏移量或一个间接定位到目标的句柄;
   4. 解析动作主要针对类或接口,字段,类方法,接口方法,方法类型等,对应常量池中的`CONSTANT_Class_info`,`CONSTANT_Fieldref_info`,`CONSTANT_Methodref_info`等;

### 3.3.3.加载过程三:Initialization

* **初始化阶段就是执行类构造器方法`<clinit>()`的过程,代码及字节码如下,此方法不需要定义,是javac编译器自动收集类中的所有类变量(什么叫做类变量?就是类的静态变量,静态变量属于类,不属于对象)的赋值动作和静态代码块中的语句合并而来,如果没有这样的动作,那么就不会有`<clinit>()`**;

~~~java
public class ByteCodeTest {

    private static String hello = "HELLO";

    static {
        hello = "world";
    }

    public static void main(String[] args) {
        System.out.println(hello);
    }
}
~~~

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801090757906.png" alt="image-20210801090757906" style="zoom: 50%;" />

* 构造器方法中指令按语句在源文件中出现的顺序执行.代码如下

~~~java
public class ByteCodeTest {

    private static String hello = "1";

    static {
        hello = "2";
        world = "8";
        System.out.println(hello);//2
        //System.out.println(world);//非法的前项引用
    }

    private static String world = "9";

    public static void main(String[] args) {
        System.out.println(hello);//2
        System.out.println(world);//9
    }
}
~~~

* `<clinit>()`不同于类的构造器.构造器是虚拟机视角下的`<init>()`;
* 若该类具有父类,JVM会保证子类的`<clint>()`执行前,父类的`<clint>()`已经执行完毕;
* 虚拟机必须保证一个类的`<clint>()`方法在多线程下被同步加锁;

## 3.4.类加载器分类

~~~java
public class ClassLoaderTest {
    public static void main(String[] args) {

        //获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);//sun.misc.Launcher$AppClassLoader@18b4aac2

        //获取其上层：扩展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader);//sun.misc.Launcher$ExtClassLoader@1540e19d

        //获取其上层：获取不到引导类加载器
        ClassLoader bootstrapClassLoader = extClassLoader.getParent();
        System.out.println(bootstrapClassLoader);//null

        //对于用户自定义类来说：默认使用系统类加载器进行加载
        ClassLoader classLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(classLoader);//sun.misc.Launcher$AppClassLoader@18b4aac2

        //String类使用引导类加载器进行加载的。---> Java的核心类库都是使用引导类加载器进行加载的。
        ClassLoader classLoader1 = String.class.getClassLoader();
        System.out.println(classLoader1);//null

    }
}
~~~

需要注意的是:**这三个类加载并不是继承关系**.

### 3.4.1.引导类加载器 BootStrap ClassLoader

* 这个类加载使用C/C++语言实现的,嵌套在JVM内部;
* 它用来加载Java的核心库(JAVA_HOME/jre/lib/rt.jar,resources.jar,sun.boot.class.path路径下的内容),用于提供JVM自身需要的类;
* 并不继承java.lang.ClassLoader(其余两种类加载器都继承于它),没有父加载器;
* 引导类加载器加载扩展类和应用程序类加载器,并指定为他们的父类加载器;
* 出于安全考虑,Bootstrap启动类加载器只加载包含名为java,javax,sun等开头的类;

### 3.4.2.扩展类加载器 Extension ClassLoader

* Java语言编写,`sun.misc.Launcher$ExtClassLoader`实现;
* 继承于ClassLoader类;
* 父类加载器为启动类加载器;
* 从`java.ext.dirs`系统属性所指定的目录中加载类库,或从JDK的安装目录`jre/lib/ext`子目录(扩展目录)下加载类库,如果用户创建JAR放在此目录下,也会自动由扩展类加载器加载;

### 3.4.3.应用类加载器 Application ClassLoader

* Java语言编写,`sun.misc.Launcher$AppClassLoader@18b4aac2`实现;
* 继承于ClassLoader类;
* 父类加载器为扩展类加载器;
* 它负责加载环境变量classpath或系统属性`java.class.path`指定路径下的类库;
* 该类加载是程序中默认的类加载器;

### 3.4.4.通过代码展示类加载器加载的范围

~~~java
public class ClassLoaderTest {
    public static void main(String[] args) {
        System.out.println("**********启动类加载器**************");
        //获取BootstrapClassLoader能够加载的api的路径
        URL[] urLs = sun.misc.Launcher.getBootstrapClassPath().getURLs();
        for (URL element : urLs) {
            System.out.println(element.toExternalForm());
        }
        /**
         * 上述结果
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/resources.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/rt.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/sunrsasign.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/jsse.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/jce.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/charsets.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/jfr.jar
         * file:/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/classes
         */

        System.out.println("***********扩展类加载器*************");
        String extDirs = System.getProperty("java.ext.dirs");
        for (String path : extDirs.split(":")) {
            System.out.println(path);
        }
        /** 
         * 上述结果
         * /Users/fechinchu/Library/Java/Extensions
         * /Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home/jre/lib/ext
         * /Library/Java/Extensions
         * /Network/Library/Java/Extensions
         * /System/Library/Java/Extensions
         * /usr/lib/java
         */
    }
}
~~~

### 3.4.5.自定义类加载器

~~~java
public class CustomClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {

        try {
            byte[] result = getClassFromCustomPath(name);
            if(result == null){
                throw new FileNotFoundException();
            }else{
                return defineClass(name,result,0,result.length);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }

        throw new ClassNotFoundException(name);
    }

    private byte[] getClassFromCustomPath(String name){
        //从自定义路径中加载指定类:细节略
        //如果指定路径的字节码文件进行了加密，则需要在此方法中进行解密操作。
        return null;
    }

    public static void main(String[] args) {
        CustomClassLoader customClassLoader = new CustomClassLoader();
        try {
            Class<?> clazz = Class.forName("One",true,customClassLoader);
            Object obj = clazz.newInstance();
            System.out.println(obj.getClass().getClassLoader());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
~~~

用户自定义类加载器实现步骤

1. 继承`java.lang.ClassLoader`类的方式,实现自己的类加载器;
2. JDK1.2.之前,继承ClassLoader类并重写`loadClass()`方法,JDK1.2.之后,建议把自定义类的加载逻辑写在`findClass()`中;
3. 编写自定义加载器时,如果没有太过于复杂的需求,直接继承`URLClassLoader`类

## 3.5.双亲委派机制

 Java虚拟机对class文件采用的是按需加载的方式,也就是说当需要使用该类时才会将它的class文件加载到内存生成class对象,而且加载某个类class文件时,Java虚拟机采用的是双亲委派模式,即把请求交由父类处理,它是一种任务委派模式;

### 3.5.1.为什么要使用双亲委派机制

加入我们在自己的项目代码中创建`java.lang.String`,并使用该了String(自定义的).实际上我们是不能去使用该类的.双亲委派机制就是为了保证核心类库的作用.这就是沙箱安全机制;

优势如下:

* 避免类的重复加载;
* 保护程序安全,防止核心API被随意篡改;

### 3.5.2.双亲委派机制工作原理

![image-20210801115143301](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210801115143301.png)

1. 如果一个类加载器收到了类加载请求,它不会自己先去加载,而是把这个请求委托给父类的加载器去执行;
2. 如果父类加载器还存在其父类加载器,则进一步向上委托,请求最终到达顶层的启动类加载器;
3. 如果父类加载器可以完成类加载任务,就成功返回,倘若父类加载器无法完成此加载任务,子类加载器才会尝试自己去加载,这就是双亲委派模式;

~~~java
package java.lang;

public class String {

    static{
        System.out.println("我是自定义的String类的静态代码块");
    }
    //错误: 在类 java.lang.String 中找不到 main 方法
    //否则JavaFX应用程序类必须扩展javafx.application.Application
    public static void main(String[] args) {
        System.out.println("hello,String");
    }
}

~~~

上述代码中,自定义`java.lang.String`执行`main()`时,会报错;

# 







